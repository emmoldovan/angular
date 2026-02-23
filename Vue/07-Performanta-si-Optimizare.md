# Performanță și Optimizare în Vue 3 (Interview Prep - Senior Frontend Architect)

> Virtual DOM, lazy loading, code splitting, v-memo, v-once, defineAsyncComponent,
> Vapor Mode (Vue 3.6+). Paralele cu Angular Change Detection, OnPush, Signals.
> Documentul acoperă toate tehnicile de optimizare relevante pentru un Senior Architect
> care lucrează pe aplicații Vue 3 de scară enterprise cu Module Federation.

---

## Cuprins

1. [Virtual DOM - Cum funcționează](#1-virtual-dom---cum-funcționează)
2. [Rendering Pipeline în Vue 3](#2-rendering-pipeline-în-vue-3)
3. [Optimizări Template (v-memo, v-once, v-cloak)](#3-optimizări-template-v-memo-v-once-v-cloak)
4. [defineAsyncComponent - Lazy Loading componente](#4-defineasynccomponent---lazy-loading-componente)
5. [Code Splitting cu Vite și Webpack](#5-code-splitting-cu-vite-și-webpack)
6. [Lazy Loading Routes](#6-lazy-loading-routes)
7. [List Rendering Optimization (v-for + key)](#7-list-rendering-optimization-v-for--key)
8. [shallowRef și shallowReactive (performance tuning)](#8-shallowref-și-shallowreactive-performance-tuning)
9. [KeepAlive - Cache componente](#9-keepalive---cache-componente)
10. [Vapor Mode (Vue 3.6+) - Fără Virtual DOM](#10-vapor-mode-vue-36---fără-virtual-dom)
11. [Profiling și Debugging Performanță](#11-profiling-și-debugging-performanță)
12. [Paralela: Angular Change Detection vs Vue Reactivity](#12-paralela-angular-change-detection-vs-vue-reactivity)
13. [Întrebări de interviu](#13-întrebări-de-interviu)

---

## 1. Virtual DOM - Cum funcționează

### Ce este Virtual DOM?

**Virtual DOM** (VDOM) este o reprezentare JavaScript **în memorie** a DOM-ului real.
În loc să manipuleze direct DOM-ul browser-ului (operație costisitoare), Vue creează
un **arbore de VNode-uri** (Virtual Nodes) care descriu structura UI-ului. Când starea
se schimbă, Vue creează un nou arbore VNode, îl compară cu cel anterior (proces numit
**patching** sau **diffing**) și aplică doar diferențele minime pe DOM-ul real.

**Concepte cheie:**

- **VNode** - un obiect JavaScript care descrie un element DOM (tag, props, children)
- **Render function** - funcție care returnează un arbore de VNode-uri
- **Patch algorithm** - algoritmul care compară două arbori VNode și generează operații DOM minime
- **Compiler** - transformă template-urile `.vue` în render functions optimizate

### Diagrama completă a fluxului

```
Template (.vue SFC)
    │
    ▼
┌─────────────────────────────────────────┐
│  Compiler (la build time)               │
│  - Analizează template-ul               │
│  - Identifică noduri statice vs dinamice│
│  - Generează render function cu         │
│    optimization hints (patch flags)     │
│  - Static Hoisting                      │
│  - Tree Flattening                      │
└─────────────────────────────────────────┘
    │
    ▼
Render Function (generată de compiler)
    │ Apelată la mount + la fiecare reactive change
    ▼
┌─────────────────────────────────────────┐
│  VNode Tree (Virtual DOM)               │
│  - Arbore de obiecte JavaScript         │
│  - Conține patch flags pe noduri        │
│  - Nodurile statice sunt hoisted        │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│  Patch Algorithm (diff)                 │
│  - Compară new VNode tree cu old tree   │
│  - Folosește patch flags → skip static  │
│  - Folosește block tree → flat diff     │
│  - Generează operații DOM minime        │
└─────────────────────────────────────────┘
    │
    ▼
Minimal DOM Updates (doar elementele schimbate)
```

### Structura unui VNode

```typescript
// Structura simplificată a unui VNode în Vue 3
interface VNode {
  type: string | Component    // 'div', 'span', sau un component
  props: Record<string, any> | null
  children: string | VNode[] | null
  key: string | number | null
  patchFlag: number           // Optimization hint de la compiler
  dynamicProps: string[]      // Lista de props dinamice
  shapeFlag: number           // Tipul VNode-ului (element, component, etc.)
  el: Element | null          // Referință la elementul DOM real (după mount)
}

// Exemplu concret
const vnode: VNode = {
  type: 'div',
  props: { class: 'container', id: 'app' },
  children: [
    { type: 'h1', props: null, children: 'Hello', patchFlag: 0 },
    { type: 'p', props: null, children: dynamicText, patchFlag: 1 } // TEXT flag
  ],
  patchFlag: 0,
  shapeFlag: 17 // ELEMENT + ARRAY_CHILDREN
}
```

### Patch Flags - Optimizări la nivel de compiler

Vue 3 compiler analizează template-ul și adaugă **patch flags** numerice pe VNode-uri.
Aceste flags spun runtime-ului **exact ce poate să se schimbe** pe fiecare nod, astfel
încât algoritmul de patch poate sări peste verificările inutile.

```typescript
// Toate Patch Flags din Vue 3
export const enum PatchFlags {
  TEXT = 1,            // Doar conținutul text se poate schimba
  CLASS = 2,           // Doar class binding-ul se poate schimba
  STYLE = 4,           // Doar style binding-ul se poate schimba
  PROPS = 8,           // Props non-class/non-style se pot schimba
  FULL_PROPS = 16,     // Props cu chei dinamice (ex: v-bind="obj")
  NEED_HYDRATION = 32, // Necesită hydration (SSR)
  STABLE_FRAGMENT = 64,// Fragment cu copii stabili (nu se reordonează)
  KEYED_FRAGMENT = 128,// Fragment cu copii keyed
  UNKEYED_FRAGMENT = 256, // Fragment cu copii fără key
  NEED_PATCH = 512,    // Component care necesită non-props patch
  DYNAMIC_SLOTS = 1024,// Component cu sloturi dinamice
  DEV_ROOT_FRAGMENT = 2048, // Dev only
  HOISTED = -1,        // Nod static hoisted (niciodată re-patched)
  BAIL = -2            // Bail out of optimization
}
```

```vue
<!-- Template Vue -->
<template>
  <div class="static-class">
    <p>Text static - nu se schimbă niciodată</p>
    <p :class="dynamicClass">{{ dynamicText }}</p>
    <span :style="{ color: textColor }">Styled</span>
    <input v-bind="dynamicAttrs" />
  </div>
</template>

<!-- Ce generează compiler-ul: -->
<!--
  <p>Text static</p>                → PatchFlag: HOISTED (-1) → skip complet
  <p :class="dyn">{{ text }}</p>    → PatchFlag: TEXT | CLASS (3) → check doar text + class
  <span :style="{ color }">        → PatchFlag: STYLE (4) → check doar style
  <input v-bind="obj" />           → PatchFlag: FULL_PROPS (16) → check toate props
-->
```

### Static Hoisting

**Static hoisting** este o optimizare la nivel de compiler unde nodurile VNode care sunt
complet statice (nu au binding-uri dinamice) sunt **extrase în afara** funcției render.
Astfel, ele sunt create o singură dată și reutilizate la fiecare re-render.

```javascript
// FĂRĂ static hoisting (ce ar genera un compiler naiv)
function render() {
  return createVNode('div', null, [
    createVNode('p', { class: 'footer' }, 'Copyright 2024'),  // recreat la fiecare render
    createVNode('p', { class: 'footer' }, 'All rights reserved'),
    createVNode('p', null, ctx.message)  // dinamic
  ])
}

// CU static hoisting (ce generează Vue 3 compiler)
const _hoisted_1 = /*#__PURE__*/ createVNode(
  'p', { class: 'footer' }, 'Copyright 2024', -1 /* HOISTED */
)
const _hoisted_2 = /*#__PURE__*/ createVNode(
  'p', { class: 'footer' }, 'All rights reserved', -1 /* HOISTED */
)

function render() {
  return createVNode('div', null, [
    _hoisted_1,                              // reutilizat, nu recreat
    _hoisted_2,                              // reutilizat, nu recreat
    createVNode('p', null, ctx.message)      // doar acesta se recreează
  ])
}
```

**Beneficii:**
- Nodurile hoisted nu sunt niciodată recreate sau comparate
- Reduce GC (Garbage Collection) pressure
- Reduce memory allocations per render cycle
- Pentru template-uri cu mult conținut static, diferența e semnificativă

### Block Tree și Tree Flattening

Vue 3 introduce conceptul de **block tree** - o structură plată care conține doar
nodurile dinamice dintr-un subtree. În loc să facă diff recursiv pe tot arborele,
algoritmul de patch compară doar nodurile dintr-un **dynamic children array** plat.

```javascript
// Template:
// <div>                          ← block root
//   <p>Static text</p>          ← static, ignored
//   <p>More static</p>          ← static, ignored
//   <p>{{ dynamic1 }}</p>        ← dynamic → tracked in block
//   <div>
//     <span>Static nested</span> ← static, ignored
//     <span>{{ dynamic2 }}</span>← dynamic → tracked in block
//   </div>
// </div>

// Block-ul root conține un flat array cu DOAR nodurile dinamice:
const block = {
  type: 'div',
  dynamicChildren: [
    { type: 'p', children: dynamic1, patchFlag: 1 },     // TEXT
    { type: 'span', children: dynamic2, patchFlag: 1 }   // TEXT
  ]
  // Nu mai face diff recursiv pe toți copiii!
}
```

**Impactul este enorm:** Dacă ai 100 de noduri în template dar doar 3 sunt dinamice,
algoritmul de patch verifică doar 3 noduri, nu 100.

### Paralela cu Angular

| Aspect | Vue 3 | Angular |
|--------|-------|---------|
| **Strategia de update** | Virtual DOM cu patch flags | Direct DOM bindings via Change Detection |
| **Overhead de bază** | VNode creation + diffing | Zone.js event interception |
| **Optimizare compiler** | Patch flags, static hoisting, block tree | Ivy compiler, incremental DOM |
| **Ce se verifică** | Doar noduri cu patch flags > 0 | Toate binding-urile din component template |
| **Skip mechanism** | Automatic via patch flags | Manual via OnPush strategy |
| **Direcția modernă** | Vapor Mode (eliminare VDOM) | Signals (eliminare Zone.js) |

> **Insight pentru interviu:** Angular Change Detection verifică **toate binding-urile**
> din template-ul unui component la fiecare CD cycle (decât cu OnPush). Vue 3 compiler
> marchează **exact** ce se poate schimba, deci runtime-ul face mult mai puțină muncă
> by default. Aceasta este diferența fundamentală de abordare.

---

## 2. Rendering Pipeline în Vue 3

### Fluxul complet de rendering

```
                    ┌──────────────┐
                    │  .vue SFC    │
                    │  Template    │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │   Compiler   │ (build time via Vite/Webpack)
                    │ @vue/compiler│
                    └──────┬───────┘
                           │ Generează optimized render function
                    ┌──────▼───────┐
                    │    Render    │ (runtime)
                    │   Function   │
                    └──────┬───────┘
                           │ Returnează VNode tree
                    ┌──────▼───────┐
                    │  VNode Tree  │
                    │ (Virtual DOM)│
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │ First Mount             │ Subsequent Updates
              │                         │
       ┌──────▼───────┐         ┌──────▼───────┐
       │    Mount     │         │    Patch     │
       │ (create DOM) │         │ (diff + apply)│
       └──────┬───────┘         └──────┬───────┘
              │                        │
       ┌──────▼───────┐         ┌──────▼───────┐
       │   Real DOM   │         │ Minimal DOM  │
       │   Created    │         │   Updates    │
       └──────────────┘         └──────────────┘
```

### Reactive Trigger → Re-render

```typescript
// Fiecare component are propria instanță de reactive effect
import { ref, watchEffect } from 'vue'

const count = ref(0)

// Intern, Vue creează un ReactiveEffect pentru fiecare component
// Echivalent conceptual:
const componentEffect = new ReactiveEffect(() => {
  // render function a componentei
  const vnode = render()
  // patch(oldVNode, newVNode)
  patch(prevTree, vnode, container)
  prevTree = vnode
})

// Când count.value se schimbă:
// 1. Proxy setter notifică effect-ul
// 2. Effect-ul este programat (nu rulat sincron!) via scheduler
// 3. Scheduler folosește queueJob + nextTick (microtask)
// 4. Re-render-ul se face în batch (o singură dată per tick)
```

### Scheduler și Batching

```typescript
import { ref, nextTick } from 'vue'

const count = ref(0)

// Toate aceste modificări vor genera UN SINGUR re-render
count.value = 1
count.value = 2
count.value = 3
// Vue batch-uiește update-urile! DOM-ul se actualizează o singură dată cu valoarea 3

// Dacă ai nevoie să accesezi DOM-ul după update:
async function incrementAndRead() {
  count.value++

  // DOM-ul NU e încă actualizat aici!
  console.log(document.getElementById('counter')?.textContent) // valoarea veche

  // Așteaptă flush-ul
  await nextTick()

  // Acum DOM-ul e actualizat
  console.log(document.getElementById('counter')?.textContent) // valoarea nouă
}
```

### Component-level Reactivity

```vue
<!-- Parent.vue -->
<template>
  <div>
    <p>{{ parentData }}</p>          <!-- Schimbarea lui parentData re-renderează Parent -->
    <ChildA :propA="valueA" />       <!-- Re-render doar dacă valueA se schimbă -->
    <ChildB />                       <!-- NU se re-renderează dacă parentData schimbă! -->
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import ChildA from './ChildA.vue'
import ChildB from './ChildB.vue'

const parentData = ref('hello')
const valueA = ref(42)

// Schimbarea lui parentData NU re-renderează ChildB
// Aceasta este o diferență majoră față de React!
// În React, Parent re-render → TOȚI copiii re-render (fără React.memo)
// În Vue, fiecare component trackuiește propriile dependențe reactive
</script>
```

### Paralela cu Angular

> **Key insight:** În Angular Default strategy, schimbarea oricărei valori în parent
> triggăruiește Change Detection pe **toți** copiii din subtree. Cu OnPush, Angular
> verifică un component doar dacă input-urile se schimbă (prin referință) sau dacă
> apare un event/async în acel component. Vue face acest lucru **automat** - fiecare
> component re-renderează doar când dependențele sale reactive se schimbă, fără
> configurare explicită.

---

## 3. Optimizări Template (v-memo, v-once, v-cloak)

### v-once - Render o singură dată

`v-once` marchează un element sau un component să fie rendat **o singură dată**.
După primul render, elementul este tratat ca conținut static și skip-uit complet
la re-render-uri viitoare.

```vue
<template>
  <!-- Cazul 1: Element simplu -->
  <h1 v-once>{{ title }}</h1>
  <!-- title se evaluează O SINGURĂ DATĂ la mount, apoi niciodată -->

  <!-- Cazul 2: Component întreg -->
  <ExpensiveChart v-once :data="initialData" />
  <!-- Component-ul NU se va re-monta niciodată, chiar dacă initialData schimbă -->

  <!-- Cazul 3: Subtree complet -->
  <div v-once>
    <h2>{{ computedTitle }}</h2>
    <p>{{ computedDescription }}</p>
    <ComplexComponent :config="staticConfig" />
  </div>
  <!-- Tot subtree-ul este skip-uit la re-render -->
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'

const title = ref('Dashboard Analytics')
const initialData = ref([/* large dataset */])

// ATENȚIE: Dacă schimbi title.value, <h1> NU se va actualiza!
// v-once este potrivit DOAR pentru date care nu se schimbă niciodată
</script>
```

**Când să folosești v-once:**
- Conținut care se setează o dată la mount și nu mai schimbă (titluri statice, copyright)
- Componente costisitoare care nu trebuie actualizate
- Template partials cu multe calcule dar date stabile

**Când NU trebuie folosit:**
- Date care se pot schimba (evident, dar e o greșeală frecventă)
- Componente cu stare internă care trebuie actualizată

### v-memo (Vue 3.2+) - Memoizare granulară

`v-memo` este o directivă care **memoizează** un subtree de template. Re-renderul
acelui subtree se face **doar** când una dintre dependențele specificate se schimbă.
Este echivalentul conceptual al `React.memo` dar la nivel de template, nu component.

```vue
<template>
  <!-- Cazul 1: Memoizare pe un element -->
  <div v-memo="[item.id, item.selected]">
    <!-- Acest div se re-renderează DOAR când item.id SAU item.selected se schimbă -->
    <h3>{{ item.name }}</h3>
    <p>{{ item.description }}</p>
    <span>{{ item.details }}</span>
    <ComplexBadge :status="item.status" />
    <!-- Chiar dacă item.name, item.description, item.status se schimbă,
         re-renderul NU se declanșează dacă id și selected sunt la fel -->
  </div>

  <!-- Cazul 2: Optimizare listă mare -->
  <div
    v-for="item in hugeList"
    :key="item.id"
    v-memo="[item.id === selectedId]"
  >
    <!-- Re-render doar dacă starea de selecție a item-ului se schimbă -->
    <div :class="{ selected: item.id === selectedId }">
      {{ item.name }}
    </div>
  </div>

  <!-- Cazul 3: v-memo cu array gol = v-once -->
  <div v-memo="[]">
    <!-- Echivalent cu v-once - nu se re-renderează niciodată -->
    {{ staticContent }}
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'

interface ListItem {
  id: number
  name: string
  description: string
  details: string
  status: string
  selected: boolean
}

const hugeList = ref<ListItem[]>([/* 10,000 items */])
const selectedId = ref<number>(1)
</script>
```

### Exemplu practic: Selecție în tabel mare

```vue
<template>
  <div class="data-table">
    <div class="table-header">
      <input
        type="checkbox"
        :checked="allSelected"
        @change="toggleAll"
      />
      <span>Selectează tot ({{ selectedCount }}/{{ items.length }})</span>
    </div>

    <!-- Fără v-memo: 10,000 rânduri se re-renderează la fiecare selecție -->
    <!-- Cu v-memo: doar rândul selectat/deselectat se re-renderează -->
    <div
      v-for="item in items"
      :key="item.id"
      v-memo="[item.id === selectedId, selectedIds.has(item.id)]"
      class="table-row"
      :class="{
        active: item.id === selectedId,
        checked: selectedIds.has(item.id)
      }"
      @click="selectItem(item.id)"
    >
      <input
        type="checkbox"
        :checked="selectedIds.has(item.id)"
        @change.stop="toggleItem(item.id)"
      />
      <span class="name">{{ item.name }}</span>
      <span class="email">{{ item.email }}</span>
      <span class="role">{{ item.role }}</span>
      <ExpensiveCellRenderer :data="item.complexData" />
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'

const items = ref<Array<{ id: number; name: string; email: string; role: string; complexData: any }>>([])
const selectedId = ref<number | null>(null)
const selectedIds = ref<Set<number>>(new Set())

const selectedCount = computed(() => selectedIds.value.size)
const allSelected = computed(() => selectedIds.value.size === items.value.length)

function selectItem(id: number) {
  selectedId.value = id
}

function toggleItem(id: number) {
  const newSet = new Set(selectedIds.value)
  if (newSet.has(id)) {
    newSet.delete(id)
  } else {
    newSet.add(id)
  }
  selectedIds.value = newSet
}

function toggleAll() {
  if (allSelected.value) {
    selectedIds.value = new Set()
  } else {
    selectedIds.value = new Set(items.value.map(i => i.id))
  }
}
</script>
```

### v-cloak - Ascunde template-ul neprocesat

`v-cloak` este folosit pentru a ascunde template-urile Vue înainte ca instanța
să fie montată. Previne **flash of uncompiled content** (FOUC) - utilizatorul
vede `{{ message }}` înainte ca Vue să proceseze template-ul.

```vue
<template>
  <!-- v-cloak este eliminat automat după mount -->
  <div v-cloak>
    <h1>{{ pageTitle }}</h1>
    <p>{{ content }}</p>
  </div>
</template>
```

```css
/* Necesar în CSS global */
[v-cloak] {
  display: none;
}

/* Alternativ, cu tranziție */
[v-cloak] {
  opacity: 0;
}
[v-cloak] + * {
  transition: opacity 0.3s;
}
```

**Notă:** `v-cloak` este relevant mai ales pentru aplicații **fără** build step
(script tag direct) sau în cazuri SSR cu hydration delay. În aplicații SFC cu
Vite/Webpack, template-urile sunt pre-compilate și `v-cloak` nu e de obicei necesar.

### Paralela cu Angular

| Directivă Vue | Echivalent Angular | Notă |
|---------------|-------------------|------|
| `v-once` | Nu există echivalent direct | Angular re-evaluează binding-urile la fiecare CD cycle |
| `v-memo` | `OnPush` strategy | OnPush e la nivel component; v-memo e la nivel template fragment |
| `v-cloak` | Nu e necesar | Angular nu expune template-uri necompilate |

> **Insight:** În Angular, poți obține ceva similar cu `v-memo` folosind OnPush +
> `ChangeDetectorRef.detach()` + `markForCheck()`, dar e mult mai verbose. Vue oferă
> aceste optimizări la nivel de template, ceea ce le face mai ușor de aplicat.

---

## 4. defineAsyncComponent - Lazy Loading componente

### Sintaxă simplă

```typescript
import { defineAsyncComponent } from 'vue'

// Cea mai simplă formă - doar un dynamic import
const AsyncChart = defineAsyncComponent(() =>
  import('./components/HeavyChart.vue')
)

// Folosire în template - identic cu un component normal
// <AsyncChart :data="chartData" />
```

### Sintaxă completă cu toate opțiunile

```typescript
import { defineAsyncComponent, h } from 'vue'
import LoadingSpinner from './LoadingSpinner.vue'
import ErrorDisplay from './ErrorDisplay.vue'

const AsyncDashboard = defineAsyncComponent({
  // Funcția loader - returnează un Promise<Component>
  loader: () => import('./components/Dashboard.vue'),

  // Component afișat în timpul loading-ului
  loadingComponent: LoadingSpinner,

  // Component afișat la error
  errorComponent: ErrorDisplay,

  // Delay (ms) înainte de a afișa loadingComponent
  // Previne flash pentru încărcări rapide
  delay: 200,

  // Timeout (ms) - dacă e depășit, se afișează errorComponent
  timeout: 10000,

  // Dacă component-ul este wrappuit în <Suspense>,
  // setează false pentru a gestiona loading/error local
  suspensible: false,

  // Handler de error cu retry logic
  onError(error, retry, fail, attempts) {
    if (error.message.match(/fetch/) && attempts <= 3) {
      // Retry la erori de rețea, maxim 3 încercări
      retry()
    } else {
      // Renunță pentru alte tipuri de erori
      fail()
    }
  }
})
```

### Exemplu practic: Conditional Heavy Components

```vue
<template>
  <div class="analytics-page">
    <div class="tab-bar">
      <button
        v-for="tab in tabs"
        :key="tab.id"
        :class="{ active: activeTab === tab.id }"
        @click="activeTab = tab.id"
      >
        {{ tab.label }}
      </button>
    </div>

    <div class="tab-content">
      <!-- Fiecare tab este un async component - se încarcă doar la nevoie -->
      <AsyncOverview v-if="activeTab === 'overview'" :dateRange="dateRange" />
      <AsyncCharts v-if="activeTab === 'charts'" :data="analyticsData" />
      <AsyncTable v-if="activeTab === 'table'" :rows="tableData" />
      <AsyncExport v-if="activeTab === 'export'" :config="exportConfig" />
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, defineAsyncComponent } from 'vue'
import LoadingSpinner from '@/components/ui/LoadingSpinner.vue'
import ErrorFallback from '@/components/ui/ErrorFallback.vue'

// Factory function pentru async components cu config standard
function createAsyncPage(loader: () => Promise<any>) {
  return defineAsyncComponent({
    loader,
    loadingComponent: LoadingSpinner,
    errorComponent: ErrorFallback,
    delay: 150,
    timeout: 15000,
    onError(error, retry, fail, attempts) {
      if (attempts <= 2) {
        retry()
      } else {
        fail()
      }
    }
  })
}

const AsyncOverview = createAsyncPage(() => import('./tabs/OverviewTab.vue'))
const AsyncCharts = createAsyncPage(() => import('./tabs/ChartsTab.vue'))
const AsyncTable = createAsyncPage(() => import('./tabs/DataTableTab.vue'))
const AsyncExport = createAsyncPage(() => import('./tabs/ExportTab.vue'))

interface Tab {
  id: string
  label: string
}

const tabs: Tab[] = [
  { id: 'overview', label: 'Overview' },
  { id: 'charts', label: 'Charts' },
  { id: 'table', label: 'Data Table' },
  { id: 'export', label: 'Export' },
]

const activeTab = ref('overview')
const dateRange = ref({ start: new Date(), end: new Date() })
const analyticsData = ref([])
const tableData = ref([])
const exportConfig = ref({})
</script>
```

### defineAsyncComponent cu Suspense

```vue
<template>
  <Suspense>
    <!-- Main content (async) -->
    <template #default>
      <AsyncDashboard :userId="userId" />
    </template>

    <!-- Fallback afișat în timpul loading-ului -->
    <template #fallback>
      <div class="loading-container">
        <SkeletonDashboard />
        <p>Se încarcă dashboard-ul...</p>
      </div>
    </template>
  </Suspense>
</template>

<script setup lang="ts">
import { defineAsyncComponent, ref, onErrorCaptured } from 'vue'
import SkeletonDashboard from './SkeletonDashboard.vue'

// Când suspensible: true (default), Suspense controlează loading state
const AsyncDashboard = defineAsyncComponent({
  loader: () => import('./Dashboard.vue'),
  suspensible: true  // default
})

const userId = ref('user-123')

// Error handling la nivel de parent
onErrorCaptured((error) => {
  console.error('Async component failed:', error)
  return false // previne propagarea
})
</script>
```

### Paralela cu Angular

```typescript
// Angular - Lazy loading component (standalone)
// În Angular, lazy loading de componente se face cel mai des prin routing:

const routes: Routes = [
  {
    path: 'dashboard',
    loadComponent: () => import('./dashboard/dashboard.component')
      .then(m => m.DashboardComponent)
  }
]

// Sau programatic cu ViewContainerRef (mai rar):
@Component({ /* ... */ })
export class HostComponent {
  constructor(private vcr: ViewContainerRef) {}

  async loadDashboard() {
    const { DashboardComponent } = await import('./dashboard/dashboard.component')
    this.vcr.createComponent(DashboardComponent)
  }
}
```

| Aspect | Vue `defineAsyncComponent` | Angular `loadComponent` |
|--------|--------------------------|------------------------|
| **Loading UI** | `loadingComponent` option | Trebuie gestionat manual |
| **Error UI** | `errorComponent` option | Trebuie gestionat manual |
| **Retry logic** | Built-in `onError(retry)` | Trebuie implementat manual |
| **Delay** | `delay` option | Nu există built-in |
| **Timeout** | `timeout` option | Nu există built-in |
| **Folosire** | Oriunde (template, programatic) | Mai ales în routing |
| **Suspense** | Integrat nativ | Nu există echivalent |

---

## 5. Code Splitting cu Vite și Webpack

### Ce este Code Splitting?

**Code splitting** este tehnica de a împărți bundle-ul aplicației în mai multe
fișiere (chunks) mai mici care se încarcă la cerere. Scopul: **reduce initial
bundle size** și **îmbunătățește Time to Interactive (TTI)**.

### Vite - Automatic Code Splitting

Vite folosește Rollup pentru build-ul de producție și face code splitting automat
pentru dynamic imports.

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],

  build: {
    // Target browsers
    target: 'es2020',

    // Chunk size warnings
    chunkSizeWarningLimit: 500, // KB

    // Rollup options
    rollupOptions: {
      output: {
        // Manual chunks - controlează ce merge în ce bundle
        manualChunks: {
          // Vendor chunks - se cacheează separat
          'vendor-vue': ['vue', 'vue-router', 'pinia'],
          'vendor-charts': ['chart.js', 'd3'],
          'vendor-utils': ['lodash-es', 'date-fns'],
          'vendor-ui': ['@headlessui/vue', '@heroicons/vue'],
        },
      },
    },

    // CSS code splitting
    cssCodeSplit: true,

    // Source maps
    sourcemap: false, // off în producție pentru performance
  },
})
```

### Advanced Vite: Chunk Strategy Functions

```typescript
// vite.config.ts - strategie avansată
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks(id: string) {
          // Toate dependențele Vue core într-un chunk
          if (id.includes('node_modules/vue') ||
              id.includes('node_modules/@vue')) {
            return 'vue-core'
          }

          // UI library într-un chunk separat
          if (id.includes('node_modules/element-plus') ||
              id.includes('node_modules/@element-plus')) {
            return 'ui-library'
          }

          // Feature-based splitting
          if (id.includes('/features/admin/')) {
            return 'feature-admin'
          }
          if (id.includes('/features/analytics/')) {
            return 'feature-analytics'
          }

          // Restul node_modules
          if (id.includes('node_modules')) {
            return 'vendor'
          }
        }
      }
    }
  }
})
```

### Webpack - Code Splitting pentru Module Federation

```javascript
// webpack.config.js - relevant pentru MFE architecture
const { ModuleFederationPlugin } = require('webpack').container

module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      maxInitialRequests: 20,
      maxAsyncRequests: 20,
      minSize: 20000,      // min 20KB per chunk
      maxSize: 250000,     // max 250KB per chunk (target)
      cacheGroups: {
        // Vue framework
        vue: {
          test: /[\\/]node_modules[\\/](vue|@vue|vue-router|pinia)[\\/]/,
          name: 'chunk-vue',
          priority: 30,
          chunks: 'all',
          reuseExistingChunk: true,
        },
        // UI components
        ui: {
          test: /[\\/]node_modules[\\/](element-plus|@headlessui)[\\/]/,
          name: 'chunk-ui',
          priority: 20,
          chunks: 'all',
        },
        // Common vendor
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'chunk-vendor',
          priority: 10,
          chunks: 'all',
          reuseExistingChunk: true,
        },
        // Common code shared between routes
        common: {
          name: 'chunk-common',
          minChunks: 2,
          priority: 5,
          chunks: 'all',
          reuseExistingChunk: true,
        },
      },
    },
    // Runtime chunk separat (necesar pentru long-term caching)
    runtimeChunk: 'single',
  },
}
```

### Dynamic Imports - Triggering Code Splits

```typescript
// 1. Route-level splitting (cel mai comun)
const routes = [
  {
    path: '/dashboard',
    component: () => import('@/views/DashboardView.vue')
    // Vite/Webpack creează automat un chunk separat
  }
]

// 2. Component-level splitting
const HeavyEditor = defineAsyncComponent(
  () => import('@/components/HeavyEditor.vue')
)

// 3. Utility-level splitting
async function generateReport() {
  // Import doar când utilizatorul cere raportul
  const { generatePDF } = await import('@/utils/pdf-generator')
  return generatePDF(data)
}

// 4. Named chunks (Webpack magic comments)
const AdminModule = defineAsyncComponent(
  () => import(/* webpackChunkName: "admin" */ '@/features/admin/AdminPanel.vue')
)

// 5. Prefetch/Preload hints (Webpack)
const Analytics = defineAsyncComponent(
  () => import(
    /* webpackChunkName: "analytics" */
    /* webpackPrefetch: true */
    '@/features/analytics/AnalyticsView.vue'
  )
)
```

### Paralela cu Angular

> **Insight:** Angular CLI face code splitting automat la nivel de **rute lazy-loaded**.
> Nu ai echivalentul direct al `manualChunks` în Angular CLI fără eject, dar poți
> controla bundling-ul prin `budgets` în `angular.json`. Vite (Vue) oferă mai mult
> control manual asupra chunk strategy, ceea ce e important în context MFE.

---

## 6. Lazy Loading Routes

### Configurare de bază

```typescript
// router/index.ts
import { createRouter, createWebHistory, type RouteRecordRaw } from 'vue-router'

// NU lazy-loaded - parte din initial bundle
import MainLayout from '@/layouts/MainLayout.vue'

const routes: RouteRecordRaw[] = [
  {
    path: '/',
    component: MainLayout,
    children: [
      {
        path: '',
        name: 'home',
        // Lazy loading cu dynamic import
        component: () => import('@/views/HomeView.vue'),
      },
      {
        path: 'products',
        name: 'products',
        component: () => import('@/views/ProductsView.vue'),
      },
      {
        path: 'products/:id',
        name: 'product-detail',
        component: () => import('@/views/ProductDetailView.vue'),
        // Doar componentul se lazy-loadează; layout-ul e deja încărcat
      },
    ],
  },
  {
    path: '/admin',
    // Admin layout + toate rutele admin = chunk separat
    component: () => import('@/layouts/AdminLayout.vue'),
    meta: { requiresAuth: true },
    children: [
      {
        path: '',
        name: 'admin-dashboard',
        component: () => import('@/features/admin/DashboardView.vue'),
      },
      {
        path: 'users',
        name: 'admin-users',
        component: () => import('@/features/admin/UsersView.vue'),
      },
      {
        path: 'settings',
        name: 'admin-settings',
        component: () => import('@/features/admin/SettingsView.vue'),
      },
    ],
  },
  {
    path: '/auth',
    children: [
      {
        path: 'login',
        name: 'login',
        component: () => import('@/views/auth/LoginView.vue'),
      },
      {
        path: 'register',
        name: 'register',
        component: () => import('@/views/auth/RegisterView.vue'),
      },
    ],
  },
]

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes,
})

export default router
```

### Prefetch Strategies

```vue
<!-- Strategie 1: Prefetch on hover -->
<template>
  <nav>
    <RouterLink
      v-for="link in navLinks"
      :key="link.path"
      :to="link.path"
      @mouseenter="prefetchRoute(link.importFn)"
    >
      {{ link.label }}
    </RouterLink>
  </nav>
</template>

<script setup lang="ts">
interface NavLink {
  path: string
  label: string
  importFn: () => Promise<any>
}

const navLinks: NavLink[] = [
  { path: '/dashboard', label: 'Dashboard', importFn: () => import('@/views/DashboardView.vue') },
  { path: '/products', label: 'Products', importFn: () => import('@/views/ProductsView.vue') },
  { path: '/analytics', label: 'Analytics', importFn: () => import('@/views/AnalyticsView.vue') },
]

// Cache pentru a preveni re-prefetch
const prefetchedRoutes = new Set<string>()

function prefetchRoute(importFn: () => Promise<any>) {
  const key = importFn.toString()
  if (!prefetchedRoutes.has(key)) {
    prefetchedRoutes.add(key)
    importFn() // Declanșează descărcarea chunk-ului
  }
}
</script>
```

```typescript
// Strategie 2: Prefetch bazat pe Intersection Observer
// composables/usePrefetchOnVisible.ts
import { onMounted, onUnmounted, type Ref } from 'vue'

export function usePrefetchOnVisible(
  elementRef: Ref<HTMLElement | null>,
  importFn: () => Promise<any>
) {
  let observer: IntersectionObserver | null = null
  let prefetched = false

  onMounted(() => {
    if (!elementRef.value) return

    observer = new IntersectionObserver(
      (entries) => {
        if (entries[0].isIntersecting && !prefetched) {
          prefetched = true
          importFn()
          observer?.disconnect()
        }
      },
      { rootMargin: '200px' } // Prefetch când e la 200px de viewport
    )

    observer.observe(elementRef.value)
  })

  onUnmounted(() => {
    observer?.disconnect()
  })
}
```

```typescript
// Strategie 3: Prefetch after idle
// Încarcă toate rutele după ce browser-ul e idle
function prefetchAllRoutes() {
  if ('requestIdleCallback' in window) {
    requestIdleCallback(() => {
      import('@/views/DashboardView.vue')
      import('@/views/ProductsView.vue')
      import('@/views/AnalyticsView.vue')
    })
  }
}

// Apelează după mount-ul aplicației
onMounted(() => {
  prefetchAllRoutes()
})
```

### Route-Level Loading States

```typescript
// router/index.ts - Global loading state
import { ref } from 'vue'

export const isRouteLoading = ref(false)

router.beforeEach(() => {
  isRouteLoading.value = true
})

router.afterEach(() => {
  isRouteLoading.value = false
})

router.onError(() => {
  isRouteLoading.value = false
})
```

```vue
<!-- App.vue -->
<template>
  <div id="app">
    <TopProgressBar v-if="isRouteLoading" />
    <RouterView />
  </div>
</template>

<script setup lang="ts">
import { isRouteLoading } from '@/router'
import TopProgressBar from '@/components/ui/TopProgressBar.vue'
</script>
```

### Paralela cu Angular

```typescript
// Angular - Lazy loading routes (echivalent)
const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module')
      .then(m => m.AdminModule),
    // sau cu standalone components:
    loadComponent: () => import('./admin/admin.component')
      .then(m => m.AdminComponent)
  }
]

// Angular Preloading Strategies
@NgModule({
  imports: [
    RouterModule.forRoot(routes, {
      preloadingStrategy: PreloadAllModules // sau custom strategy
    })
  ]
})
```

| Aspect | Vue Router | Angular Router |
|--------|-----------|---------------|
| **Sintaxă** | `component: () => import()` | `loadComponent / loadChildren` |
| **Granularity** | Per component | Per module sau per component |
| **Preloading** | Manual (hover, idle, observer) | Built-in strategies |
| **Loading UI** | Manual + `defineAsyncComponent` | Manual via resolver/guard |
| **Route Guards** | `beforeEnter`, navigation guards | `canActivate`, `canLoad` |

---

## 7. List Rendering Optimization (v-for + key)

### Importanța key-urilor corecte

```vue
<!-- GREȘIT: key bazat pe index -->
<template>
  <div v-for="(item, index) in items" :key="index">
    <input v-model="item.name" />
  </div>
  <!-- Dacă ștergi un element din mijloc, Vue va refolosi DOM-ul greșit! -->
  <!-- Input-urile vor avea valori amestecate -->
</template>

<!-- CORECT: key bazat pe identificator unic stabil -->
<template>
  <div v-for="item in items" :key="item.id">
    <input v-model="item.name" />
  </div>
  <!-- Vue știe exact ce element a fost șters și actualizează corect -->
</template>
```

### Optimizări pentru liste mari

```vue
<template>
  <div class="optimized-list">
    <!-- Combinația v-for + v-memo pentru liste de 10,000+ items -->
    <div
      v-for="item in filteredItems"
      :key="item.id"
      v-memo="[item.id === activeId, item.lastModified]"
      class="list-item"
      :class="{ active: item.id === activeId }"
      @click="setActive(item.id)"
    >
      <div class="item-content">
        <strong>{{ item.title }}</strong>
        <p>{{ item.summary }}</p>
        <span class="date">{{ formatDate(item.lastModified) }}</span>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, shallowRef } from 'vue'

interface ListItem {
  id: string
  title: string
  summary: string
  lastModified: Date
  category: string
}

// shallowRef pentru lista mare - nu track-uiește nested changes
const items = shallowRef<ListItem[]>([])
const activeId = ref<string | null>(null)
const filterCategory = ref<string>('all')

// computed pentru filtrare (cached, recalculat doar la schimbare)
const filteredItems = computed(() => {
  if (filterCategory.value === 'all') return items.value
  return items.value.filter(i => i.category === filterCategory.value)
})

function setActive(id: string) {
  activeId.value = id
}

function formatDate(date: Date): string {
  return date.toLocaleDateString('ro-RO')
}
</script>
```

### Virtual Scrolling cu vue-virtual-scroller

```vue
<template>
  <div class="virtual-list-container">
    <!-- RecycleScroller: recilează DOM elements, ideal pentru liste enorme -->
    <RecycleScroller
      class="scroller"
      :items="items"
      :item-size="80"
      key-field="id"
      v-slot="{ item, index, active }"
    >
      <div class="list-item" :class="{ active: item.id === selectedId }">
        <div class="avatar">{{ item.initials }}</div>
        <div class="info">
          <strong>{{ item.name }}</strong>
          <p>{{ item.email }}</p>
        </div>
        <span class="index">#{{ index }}</span>
      </div>
    </RecycleScroller>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import { RecycleScroller } from 'vue-virtual-scroller'
import 'vue-virtual-scroller/dist/vue-virtual-scroller.css'

interface UserItem {
  id: string
  name: string
  email: string
  initials: string
}

// 100,000 items - nu ar fi posibil fără virtual scrolling
const items = ref<UserItem[]>(
  Array.from({ length: 100_000 }, (_, i) => ({
    id: `user-${i}`,
    name: `User ${i}`,
    email: `user${i}@example.com`,
    initials: `U${i}`.slice(0, 2)
  }))
)

const selectedId = ref<string | null>(null)
</script>

<style scoped>
.scroller {
  height: 600px;
  overflow-y: auto;
}

.list-item {
  height: 80px;
  display: flex;
  align-items: center;
  padding: 0 16px;
  border-bottom: 1px solid #eee;
}
</style>
```

### DynamicScroller pentru items cu dimensiuni variabile

```vue
<template>
  <DynamicScroller
    :items="messages"
    :min-item-size="54"
    key-field="id"
    class="chat-scroller"
  >
    <template v-slot="{ item, index, active }">
      <DynamicScrollerItem
        :item="item"
        :active="active"
        :data-index="index"
        :size-dependencies="[item.content, item.attachments?.length]"
      >
        <ChatMessage :message="item" />
      </DynamicScrollerItem>
    </template>
  </DynamicScroller>
</template>

<script setup lang="ts">
import { DynamicScroller, DynamicScrollerItem } from 'vue-virtual-scroller'
import ChatMessage from './ChatMessage.vue'

interface Message {
  id: string
  content: string
  attachments?: string[]
  sender: string
  timestamp: Date
}

const messages = defineProps<{ messages: Message[] }>()
</script>
```

### Pagination vs Infinite Scroll

```typescript
// composables/usePagination.ts
import { ref, computed, watch } from 'vue'

interface PaginationOptions<T> {
  fetchFn: (page: number, pageSize: number) => Promise<{ data: T[]; total: number }>
  pageSize?: number
}

export function usePagination<T>({ fetchFn, pageSize = 20 }: PaginationOptions<T>) {
  const currentPage = ref(1)
  const items = ref<T[]>([]) as Ref<T[]>
  const totalItems = ref(0)
  const isLoading = ref(false)

  const totalPages = computed(() => Math.ceil(totalItems.value / pageSize))
  const hasNextPage = computed(() => currentPage.value < totalPages.value)
  const hasPrevPage = computed(() => currentPage.value > 1)

  async function loadPage(page: number) {
    isLoading.value = true
    try {
      const result = await fetchFn(page, pageSize)
      items.value = result.data
      totalItems.value = result.total
      currentPage.value = page
    } finally {
      isLoading.value = false
    }
  }

  function nextPage() {
    if (hasNextPage.value) loadPage(currentPage.value + 1)
  }

  function prevPage() {
    if (hasPrevPage.value) loadPage(currentPage.value - 1)
  }

  // Load first page
  loadPage(1)

  return { items, currentPage, totalPages, isLoading, hasNextPage, hasPrevPage, nextPage, prevPage, loadPage }
}
```

```typescript
// composables/useInfiniteScroll.ts
import { ref, onMounted, onUnmounted } from 'vue'

interface InfiniteScrollOptions<T> {
  fetchFn: (cursor: string | null) => Promise<{ data: T[]; nextCursor: string | null }>
  rootElement?: HTMLElement | null
}

export function useInfiniteScroll<T>({ fetchFn }: InfiniteScrollOptions<T>) {
  const items = ref<T[]>([]) as Ref<T[]>
  const isLoading = ref(false)
  const hasMore = ref(true)
  const cursor = ref<string | null>(null)
  const sentinelRef = ref<HTMLElement | null>(null)
  let observer: IntersectionObserver | null = null

  async function loadMore() {
    if (isLoading.value || !hasMore.value) return

    isLoading.value = true
    try {
      const result = await fetchFn(cursor.value)
      items.value = [...items.value, ...result.data]
      cursor.value = result.nextCursor
      hasMore.value = result.nextCursor !== null
    } finally {
      isLoading.value = false
    }
  }

  onMounted(() => {
    if (!sentinelRef.value) return

    observer = new IntersectionObserver(
      (entries) => {
        if (entries[0].isIntersecting) {
          loadMore()
        }
      },
      { rootMargin: '300px' }
    )

    observer.observe(sentinelRef.value)
  })

  onUnmounted(() => {
    observer?.disconnect()
  })

  // Load initial batch
  loadMore()

  return { items, isLoading, hasMore, sentinelRef, loadMore }
}
```

### Paralela cu Angular

> **Insight:** Angular folosește `trackBy` în `*ngFor` (echivalentul `:key` din Vue).
> Fără `trackBy`, Angular re-creează toate DOM elements la fiecare schimbare a listei.
> Vue cere `:key` explicit și oferă `v-memo` ca optimizare suplimentară. Pentru
> virtual scrolling, Angular CDK oferă `cdk-virtual-scroll-viewport`, echivalentul
> `vue-virtual-scroller`.

---

## 8. shallowRef și shallowReactive (performance tuning)

### Problema: Deep Reactivity

```typescript
import { ref, reactive } from 'vue'

// ref() face deep reactive - FIECARE nested property e tracked
const deepState = ref({
  users: [
    {
      id: 1,
      name: 'Emanuel',
      address: {
        city: 'Cluj',
        street: 'Strada Exemplu',
        geo: { lat: 46.77, lng: 23.59 }
      },
      permissions: ['read', 'write', 'admin'],
      metadata: { /* 50 nested properties */ }
    },
    // ... 10,000 more users
  ],
  config: { /* deeply nested config */ },
  cache: { /* large cache object */ }
})

// Vue creează Proxy pe FIECARE nivel nested
// Asta înseamnă: 10,000 users × nested objects = zeci de mii de Proxy-uri
// Cost: memorie + overhead la acces
```

### shallowRef - Reactivity doar pe top-level

```typescript
import { shallowRef, triggerRef } from 'vue'

// shallowRef - DOAR .value assignment triggerează update
const users = shallowRef<User[]>([])

// NU triggerează re-render (nested mutation)
users.value[0].name = 'Updated Name'  // NIMIC nu se întâmplă

// Triggerează re-render (value assignment)
users.value = [...users.value]  // OK - nouă referință
users.value = users.value.map(u =>
  u.id === 1 ? { ...u, name: 'Updated' } : u
)

// Force trigger fără a schimba referința
users.value[0].name = 'Updated Name'
triggerRef(users)  // Forțează re-render manual
```

### shallowReactive - Reactivity doar pe proprietăți de top-level

```typescript
import { shallowReactive } from 'vue'

const state = shallowReactive({
  count: 0,           // reactive (top-level)
  nested: {           // NOT reactive (nested)
    value: 'hello'
  },
  items: []           // NOT reactive (nested mutations)
})

state.count++                    // Triggerează re-render
state.nested.value = 'world'     // NU triggerează re-render
state.nested = { value: 'world' }// Triggerează re-render (top-level assignment)
state.items.push('new')          // NU triggerează re-render
state.items = [...state.items, 'new'] // Triggerează re-render
```

### Exemplu practic: Large Dataset Management

```vue
<template>
  <div class="data-manager">
    <div class="stats">
      Total: {{ dataStats.total }} | Filtered: {{ dataStats.filtered }}
    </div>

    <input v-model="searchQuery" placeholder="Caută..." />

    <div
      v-for="item in displayItems"
      :key="item.id"
      v-memo="[item.id === selectedId]"
    >
      {{ item.name }} - {{ item.value }}
    </div>
  </div>
</template>

<script setup lang="ts">
import { shallowRef, ref, computed, triggerRef } from 'vue'

interface DataItem {
  id: string
  name: string
  value: number
  category: string
  metadata: Record<string, unknown> // Obiect mare, nu trebuie reactive
}

// shallowRef pentru dataset-ul mare
const rawData = shallowRef<DataItem[]>([])
const searchQuery = ref('')
const selectedId = ref<string | null>(null)

// computed funcționează normal cu shallowRef
const displayItems = computed(() => {
  const query = searchQuery.value.toLowerCase()
  if (!query) return rawData.value.slice(0, 100) // Paginare
  return rawData.value
    .filter(item => item.name.toLowerCase().includes(query))
    .slice(0, 100)
})

const dataStats = computed(() => ({
  total: rawData.value.length,
  filtered: displayItems.value.length
}))

// Funcții de update - creează referințe noi
function addItem(item: DataItem) {
  rawData.value = [...rawData.value, item]
}

function updateItem(id: string, updates: Partial<DataItem>) {
  rawData.value = rawData.value.map(item =>
    item.id === id ? { ...item, ...updates } : item
  )
}

function removeItem(id: string) {
  rawData.value = rawData.value.filter(item => item.id !== id)
}

// Bulk update cu triggerRef (performance optimization)
function bulkUpdateValues(multiplier: number) {
  for (const item of rawData.value) {
    item.value *= multiplier
  }
  triggerRef(rawData) // O singură notificare pentru toate schimbările
}

// Load data
async function loadData() {
  const response = await fetch('/api/data')
  rawData.value = await response.json() // Single assignment = single trigger
}
</script>
```

### Când să folosești shallow vs deep

| Scenariu | `ref` / `reactive` | `shallowRef` / `shallowReactive` |
|----------|--------------------|---------------------------------|
| Form state (câteva câmpuri) | Da | Nu e necesar |
| Liste mici (< 100 items) | Da | Nu e necesar |
| Liste mari (1000+ items) | Nu (performanță) | Da |
| Obiecte imutable (API responses) | Nu e necesar | Da (ideal) |
| Third-party instances (Chart.js, Map) | Nu (poate cauza probleme) | Da |
| Stare UI simplă (toggles, counters) | Da | Nu e necesar |
| Cache / store cu date mari | Nu | Da |

### Paralela cu Angular

> **Insight:** Conceptul de shallowRef e similar cu **imutabilitatea** din Angular
> OnPush. Cu OnPush, Angular verifică componente doar dacă input references se
> schimbă (shallow comparison). shallowRef forțează același pattern: schimbi referința
> pentru a triggera updates, nu mutezi in-place.

---

## 9. KeepAlive - Cache componente

### Sintaxă de bază

```vue
<template>
  <div class="app-layout">
    <nav>
      <button @click="currentView = 'ProductList'">Produse</button>
      <button @click="currentView = 'Dashboard'">Dashboard</button>
      <button @click="currentView = 'Settings'">Setări</button>
    </nav>

    <!-- KeepAlive păstrează instanțele componentelor în memorie -->
    <KeepAlive>
      <component :is="currentView" />
    </KeepAlive>
    <!-- Când user-ul navighează înapoi, componenta e restaurată exact cum era -->
  </div>
</template>

<script setup lang="ts">
import { ref, shallowRef } from 'vue'
import ProductList from './ProductList.vue'
import Dashboard from './Dashboard.vue'
import Settings from './Settings.vue'

const components = { ProductList, Dashboard, Settings }
const currentView = shallowRef(ProductList)
</script>
```

### Include / Exclude / Max

```vue
<template>
  <!-- Include: doar aceste componente sunt cached -->
  <KeepAlive :include="['ProductList', 'Dashboard']">
    <component :is="currentView" />
  </KeepAlive>

  <!-- Exclude: aceste componente NU sunt cached -->
  <KeepAlive :exclude="['Settings', 'Profile']">
    <component :is="currentView" />
  </KeepAlive>

  <!-- Max: LRU cache cu limită -->
  <KeepAlive :max="5">
    <component :is="currentView" />
  </KeepAlive>
  <!-- Când se depășește max, cea mai veche componentă (least recently used) e distrusă -->

  <!-- Combinații -->
  <KeepAlive :include="cachedViews" :max="10">
    <RouterView v-slot="{ Component }">
      <component :is="Component" />
    </RouterView>
  </KeepAlive>

  <!-- Regex pattern -->
  <KeepAlive :include="/^(Product|Dashboard)/">
    <component :is="currentView" />
  </KeepAlive>
</template>
```

### Lifecycle Hooks: onActivated / onDeactivated

```vue
<!-- ProductList.vue (cached cu KeepAlive) -->
<template>
  <div class="product-list">
    <div v-for="product in products" :key="product.id">
      {{ product.name }} - {{ product.price }}
    </div>
    <p v-if="lastRefreshed">
      Ultima actualizare: {{ formatTime(lastRefreshed) }}
    </p>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, onActivated, onDeactivated } from 'vue'

interface Product {
  id: string
  name: string
  price: number
}

const products = ref<Product[]>([])
const lastRefreshed = ref<Date | null>(null)
let refreshInterval: ReturnType<typeof setInterval> | null = null

// onMounted - apelat O SINGURĂ DATĂ (la prima montare)
onMounted(() => {
  console.log('ProductList mounted (prima dată)')
  fetchProducts()
})

// onActivated - apelat la FIECARE activare (inclusiv prima montare)
onActivated(() => {
  console.log('ProductList activated')

  // Verifică dacă datele sunt stale (mai vechi de 5 minute)
  if (lastRefreshed.value) {
    const age = Date.now() - lastRefreshed.value.getTime()
    if (age > 5 * 60 * 1000) {
      fetchProducts() // Refresh stale data
    }
  }

  // Pornește polling
  refreshInterval = setInterval(fetchProducts, 30_000) // 30s
})

// onDeactivated - apelat când componenta e dezactivată (ascunsă)
onDeactivated(() => {
  console.log('ProductList deactivated')

  // Oprește polling pentru a preveni network requests inutile
  if (refreshInterval) {
    clearInterval(refreshInterval)
    refreshInterval = null
  }
})

// onUnmounted - apelat DOAR dacă componenta e distrusă (KeepAlive :max exceeded)
onUnmounted(() => {
  console.log('ProductList unmounted (distrus din cache)')
  if (refreshInterval) {
    clearInterval(refreshInterval)
  }
})

async function fetchProducts() {
  const response = await fetch('/api/products')
  products.value = await response.json()
  lastRefreshed.value = new Date()
}

function formatTime(date: Date): string {
  return date.toLocaleTimeString('ro-RO')
}
</script>
```

### KeepAlive cu Vue Router

```vue
<!-- App.vue -->
<template>
  <RouterView v-slot="{ Component, route }">
    <!-- Cache doar rutele marcate cu meta.keepAlive -->
    <KeepAlive :include="cachedRouteNames" :max="10">
      <component :is="Component" :key="route.fullPath" />
    </KeepAlive>
  </RouterView>
</template>

<script setup lang="ts">
import { computed } from 'vue'
import { useRouter } from 'vue-router'

const router = useRouter()

// Extrage numele rutelor care trebuie cached
const cachedRouteNames = computed(() => {
  return router.getRoutes()
    .filter(route => route.meta?.keepAlive)
    .map(route => route.name as string)
})
</script>
```

```typescript
// router/index.ts - Route meta configuration
const routes: RouteRecordRaw[] = [
  {
    path: '/products',
    name: 'ProductList',
    component: () => import('@/views/ProductList.vue'),
    meta: { keepAlive: true }  // Cached
  },
  {
    path: '/products/:id',
    name: 'ProductDetail',
    component: () => import('@/views/ProductDetail.vue'),
    meta: { keepAlive: false } // NOT cached
  },
  {
    path: '/dashboard',
    name: 'Dashboard',
    component: () => import('@/views/Dashboard.vue'),
    meta: { keepAlive: true }  // Cached
  },
]
```

### Beneficii și riscuri KeepAlive

| Beneficii | Riscuri |
|-----------|---------|
| Scroll position păstrat | Memory leaks (prea multe instanțe cached) |
| Form state păstrat | Date stale (nu se refresh automat) |
| Nu re-fetch date la revenire | Event listeners rămân active |
| Tranziții mai rapide | Stare inconsistentă dacă backend-ul s-a schimbat |
| UX fluid (nu pierde context) | `:max` trebuie gestionat atent |

### Paralela cu Angular

> **Insight:** Angular NU are echivalent built-in pentru `KeepAlive`. Componentele
> Angular sunt distruse la navigare. Pentru a obține un comportament similar trebuie
> implementat un custom `RouteReuseStrategy`. Vue oferă acest lucru out-of-the-box
> cu o API simplă și declarativă.

---

## 10. Vapor Mode (Vue 3.6+) - Fără Virtual DOM

### Ce este Vapor Mode?

**Vapor Mode** este o nouă strategie de compilare în Vue care elimină Virtual DOM-ul
complet. În loc să genereze VNode-uri și să facă diffing, compiler-ul Vapor generează
**cod imperativ** care manipulează DOM-ul direct, similar cu Solid.js sau Svelte.

```
Modul Classic (VDOM):
  Template → Render Function → VNode Tree → Diff → DOM Updates

Modul Vapor (fără VDOM):
  Template → Compiled DOM Operations → Direct DOM Updates
  (fără VNode creation, fără diffing, fără patch algorithm)
```

### De ce Vapor Mode?

- **2-3x mai rapid** la updates (nu mai face diff)
- **Bundle mai mic** (nu include runtime-ul VDOM)
- **Memorie redusă** (nu creează VNode objects)
- **Backwards compatible** cu componente VDOM existente
- **Opt-in per component** (nu trebuie migrat totul)

### Cum arată Vapor Mode

```vue
<!-- ClassicComponent.vue (VDOM mode - default) -->
<script setup lang="ts">
import { ref } from 'vue'
const count = ref(0)
const doubled = computed(() => count.value * 2)
</script>

<template>
  <div>
    <p>Count: {{ count }}</p>
    <p>Doubled: {{ doubled }}</p>
    <button @click="count++">Increment</button>
  </div>
</template>

<!-- Ce generează compiler-ul CLASIC: -->
<!--
function render(_ctx) {
  return createVNode('div', null, [
    createVNode('p', null, 'Count: ' + _ctx.count),
    createVNode('p', null, 'Doubled: ' + _ctx.doubled),
    createVNode('button', { onClick: () => _ctx.count++ }, 'Increment')
  ])
}
La fiecare update: recreează VNode tree → diff → patch DOM
-->
```

```vue
<!-- VaporComponent.vue (Vapor mode - opt-in) -->
<script setup vapor lang="ts">
import { ref, computed } from 'vue'
const count = ref(0)
const doubled = computed(() => count.value * 2)
</script>

<template>
  <div>
    <p>Count: {{ count }}</p>
    <p>Doubled: {{ doubled }}</p>
    <button @click="count++">Increment</button>
  </div>

  <!-- IDENTIC ca template! API-ul NU se schimbă. -->
  <!-- Doar keyword-ul "vapor" în script tag activează modul. -->
</template>

<!-- Ce generează compiler-ul VAPOR (conceptual): -->
<!--
function setup() {
  const count = ref(0)
  const doubled = computed(() => count.value * 2)

  // Creare directă DOM
  const div = document.createElement('div')
  const p1 = document.createElement('p')
  const p2 = document.createElement('p')
  const btn = document.createElement('button')

  // Text nodes
  const text1 = document.createTextNode('Count: ' + count.value)
  const text2 = document.createTextNode('Doubled: ' + doubled.value)

  // Subscribe la reactive changes - update direct
  watchEffect(() => {
    text1.textContent = 'Count: ' + count.value
  })
  watchEffect(() => {
    text2.textContent = 'Doubled: ' + doubled.value
  })

  btn.addEventListener('click', () => count.value++)
  btn.textContent = 'Increment'

  // Build tree
  p1.appendChild(text1)
  p2.appendChild(text2)
  div.append(p1, p2, btn)

  return div
}
// La update: reactive effect → direct DOM mutation (fără diff!)
-->
```

### Interoperabilitate: Vapor + VDOM Components

```vue
<!-- App.vue (VDOM mode) -->
<template>
  <div>
    <ClassicHeader />           <!-- VDOM component -->
    <VaporDashboard />          <!-- Vapor component - funcționează! -->
    <ClassicFooter />           <!-- VDOM component -->
  </div>
</template>

<script setup lang="ts">
// Poți mixa componente Vapor și VDOM în aceeași aplicație
import ClassicHeader from './ClassicHeader.vue'
import VaporDashboard from './VaporDashboard.vue'  // vapor component
import ClassicFooter from './ClassicFooter.vue'
</script>
```

### Ce suportă Vapor Mode

```
Suportat:
  ✅ Composition API (ref, reactive, computed, watch, etc.)
  ✅ Template syntax (v-if, v-for, v-on, v-bind, v-model, etc.)
  ✅ Component props, emits, slots
  ✅ Lifecycle hooks (onMounted, onUnmounted, etc.)
  ✅ Provide/Inject
  ✅ Watchers
  ✅ Mixare cu componente VDOM

Limitări inițiale:
  ⚠️ Transition/TransitionGroup - suport parțial
  ⚠️ KeepAlive - suport parțial
  ⚠️ Suspense - suport parțial
  ⚠️ Custom directives - necesită adaptare
  ⚠️ Render functions (h()) - doar VDOM mode
```

### Configurare Vapor Mode

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [
    vue({
      // Activează suport Vapor Mode
      features: {
        vaporMode: true
      }
    })
  ]
})
```

### Când să folosești Vapor Mode

| Scenariu | Recomandare |
|----------|-------------|
| Componente cu update-uri foarte frecvente (realtime data) | Vapor Mode |
| Liste mari cu interacțiuni complexe | Vapor Mode |
| Componente care folosesc KeepAlive extensiv | VDOM Mode (deocamdată) |
| Componente cu tranziții complexe | VDOM Mode (deocamdată) |
| Componente noi, performance-critical | Vapor Mode |
| Componente legacy existente | VDOM Mode (nu migra fără motiv) |

### Paralela cu Angular

| Aspect | Vue Vapor Mode | Angular Signals (Zoneless) |
|--------|---------------|---------------------------|
| **Eliminare overhead** | Elimină Virtual DOM | Elimină Zone.js |
| **Update strategy** | Fine-grained reactive → direct DOM | Signal graph → direct DOM |
| **Opt-in** | Per component (`vapor` keyword) | Per application (`provideZonelessChangeDetection`) |
| **Interoperabilitate** | Vapor + VDOM coexistă | Signals + Zone.js coexistă |
| **Inspirație** | Solid.js | Angular Signals (design propriu) |
| **Bundle impact** | Reduce runtime Vue | Elimină Zone.js (~13KB) |
| **Maturitate** | Nou (Vue 3.6+) | În dezvoltare (Angular 17+) |

> **Insight pentru interviu:** Atât Vue cu Vapor Mode cât și Angular cu Signals se
> îndreaptă spre aceeași direcție: **eliminarea layer-urilor intermediare** (VDOM
> respectiv Zone.js) în favoarea **reactivity fine-grained** care face update-uri
> direct pe DOM. Aceasta este convergența naturală a framework-urilor moderne.

---

## 11. Profiling și Debugging Performanță

### Vue DevTools - Performance Tab

```typescript
// Activare performance tracking în development
// main.ts
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)

// Activează performance tracking (doar dev)
if (import.meta.env.DEV) {
  app.config.performance = true
  // Acum Vue DevTools va arăta timing-uri per component:
  // - init: timpul de inițializare
  // - render: timpul de render
  // - patch: timpul de patching
}

app.mount('#app')
```

### Chrome DevTools Performance

```typescript
// Marcaje custom pentru Chrome Performance tab
function expensiveOperation() {
  performance.mark('expensive-start')

  // ... operația costisitoare

  performance.mark('expensive-end')
  performance.measure('Expensive Operation', 'expensive-start', 'expensive-end')
}

// User Timing API în componente Vue
import { onMounted, onUpdated } from 'vue'

onMounted(() => {
  performance.mark('component-mounted')
})

onUpdated(() => {
  performance.mark('component-updated')
  performance.measure(
    'Component Update Duration',
    'component-mounted',
    'component-updated'
  )
})
```

### Bundle Analysis

```bash
# Vite - vizualizare bundle
npm install -D rollup-plugin-visualizer

# Sau cu vite-bundle-analyzer
npx vite-bundle-visualizer
```

```typescript
// vite.config.ts - cu visualizer
import { visualizer } from 'rollup-plugin-visualizer'

export default defineConfig({
  plugins: [
    vue(),
    visualizer({
      open: true,
      filename: 'dist/stats.html',
      gzipSize: true,
      brotliSize: true,
    })
  ]
})
```

### Web Vitals Monitoring

```typescript
// utils/webVitals.ts
import { onCLS, onFID, onLCP, onFCP, onTTFB, onINP } from 'web-vitals'

interface VitalMetric {
  name: string
  value: number
  rating: 'good' | 'needs-improvement' | 'poor'
}

function reportVital(metric: VitalMetric) {
  // Trimite la analytics
  console.log(`[Web Vital] ${metric.name}: ${metric.value} (${metric.rating})`)

  // Sau trimite la un endpoint de monitoring
  if (import.meta.env.PROD) {
    navigator.sendBeacon('/api/vitals', JSON.stringify(metric))
  }
}

export function initWebVitals() {
  onCLS(reportVital)   // Cumulative Layout Shift
  onFID(reportVital)   // First Input Delay (deprecated, replaced by INP)
  onINP(reportVital)   // Interaction to Next Paint
  onLCP(reportVital)   // Largest Contentful Paint
  onFCP(reportVital)   // First Contentful Paint
  onTTFB(reportVital)  // Time to First Byte
}
```

```vue
<!-- Folosire în App.vue -->
<script setup lang="ts">
import { onMounted } from 'vue'
import { initWebVitals } from '@/utils/webVitals'

onMounted(() => {
  initWebVitals()
})
</script>
```

### Performance Composable

```typescript
// composables/usePerformanceMonitor.ts
import { ref, onMounted, onUpdated, onUnmounted, getCurrentInstance } from 'vue'

export function usePerformanceMonitor(componentName?: string) {
  const renderCount = ref(0)
  const lastRenderTime = ref(0)
  const avgRenderTime = ref(0)
  const totalRenderTime = ref(0)

  const name = componentName || getCurrentInstance()?.type?.__name || 'Unknown'
  let renderStart = 0

  onMounted(() => {
    if (import.meta.env.DEV) {
      console.log(`[Perf] ${name} mounted`)
    }
  })

  onUpdated(() => {
    renderCount.value++
    const duration = performance.now() - renderStart
    lastRenderTime.value = duration
    totalRenderTime.value += duration
    avgRenderTime.value = totalRenderTime.value / renderCount.value

    if (import.meta.env.DEV && duration > 16) {
      // Mai mult de 16ms = sub 60fps
      console.warn(
        `[Perf] ${name} slow render: ${duration.toFixed(2)}ms ` +
        `(avg: ${avgRenderTime.value.toFixed(2)}ms, count: ${renderCount.value})`
      )
    }
  })

  // Hook pentru a marca începutul render-ului
  function markRenderStart() {
    renderStart = performance.now()
  }

  return {
    renderCount,
    lastRenderTime,
    avgRenderTime,
    markRenderStart
  }
}
```

### Checklist de performanță pentru Vue apps

```
Pre-deployment Performance Checklist:
═══════════════════════════════════════

Build & Bundle:
  □ Code splitting activat (route-level + component-level)
  □ Tree-shaking funcționează (import-uri named, nu default)
  □ Bundle analysis verificat (fără dependințe inutile)
  □ Vendor chunks separate (long-term caching)
  □ Gzip/Brotli compression activat pe server
  □ Source maps dezactivate în producție

Runtime:
  □ v-memo pe liste mari (1000+ items)
  □ shallowRef pentru dataset-uri mari
  □ Virtual scrolling pentru liste foarte mari (10,000+)
  □ KeepAlive pe componente frecvent navigate
  □ defineAsyncComponent pentru componente grele nefolosite la mount
  □ Computed properties (nu metode) pentru derivări costisitoare
  □ :key stabil și unic pe v-for

Images & Assets:
  □ Lazy loading images (loading="lazy" sau IntersectionObserver)
  □ Formate moderne (WebP, AVIF)
  □ Responsive images (srcset)
  □ SVG sprites sau icon components

Network:
  □ API response caching (SWR/stale-while-revalidate)
  □ Prefetch pe hover pentru rute
  □ Request deduplication
  □ Pagination sau infinite scroll (nu all-at-once)

Monitoring:
  □ Web Vitals tracking activ
  □ Error boundary components
  □ Performance budgets definite
  □ Lighthouse CI în pipeline
```

### Paralela cu Angular

> **Insight:** Angular oferă profiling prin Angular DevTools (Component Explorer +
> Profiler). Conceptele de Web Vitals, bundle analysis și Lighthouse sunt identice
> între Vue și Angular. Diferența principală: Vue nu are echivalentul `NgZone` care
> poate cauza probleme de performanță specifice Angular (excessive change detection).

---

## 12. Paralela completă: Angular Change Detection vs Vue Reactivity

### Tabela comparativă detaliată

| Aspect | Angular (Classic) | Angular (Signals) | Vue 3 (VDOM) | Vue 3 (Vapor) |
|--------|-------------------|-------------------|--------------|---------------|
| **Mecanism** | Zone.js + CD tree walk | Signal graph | Proxy-based reactivity + VDOM | Proxy + direct DOM |
| **Granularitate** | Per component | Per signal consumer | Per component (auto) | Per binding |
| **Default behavior** | Check tot tree-ul | Lazy evaluation | Check doar affected components | Direct DOM update |
| **Opt-in optimization** | OnPush (manual) | Automatic | Automatic | Automatic |
| **Skip mechanism** | OnPush + immutable refs | Signal equality check | Fine-grained dep tracking | Fine-grained dep tracking |
| **Overhead layer** | Zone.js (~13KB) | Fără Zone.js | Virtual DOM runtime | Fără VDOM |
| **Async handling** | Zone.js patches async | Signal effects | nextTick batching | nextTick batching |
| **Debugging** | Angular DevTools Profiler | Angular DevTools | Vue DevTools Performance | Vue DevTools |
| **Performance ceiling** | Foarte înalt (cu signals) | Foarte înalt | Foarte înalt | Cel mai înalt |

### Cum funcționează Change Detection în Angular (pentru context)

```typescript
// Angular - Default Change Detection
// Zone.js interceptează TOATE async operations:
// - setTimeout, setInterval
// - Promise.resolve/reject
// - addEventListener (click, input, etc.)
// - XMLHttpRequest, fetch
// - requestAnimationFrame

// La fiecare async event:
// 1. Zone.js detectează că ceva async s-a terminat
// 2. Angular pornește Change Detection de la ROOT
// 3. Parcurge FIECARE component din tree (top-down)
// 4. La fiecare component, verifică TOATE template bindings
// 5. Dacă o valoare s-a schimbat → update DOM

// Cu OnPush:
// Angular SKIP-uiește componente OnPush dacă:
// - Input references nu s-au schimbat (shallow comparison)
// - Nu a apărut un event în acel component
// - Nu s-a apelat markForCheck()

// ═══════════════════════════════════════════════
// Vue 3 - Proxy-based Reactivity
// Fiecare ref/reactive creează un Proxy
// Fiecare component are un ReactiveEffect care trackuiește ce ref-uri citește
// Când un ref se schimbă → doar componentele care îl citesc sunt notificate
// NU parcurge tot tree-ul, NU verifică toate binding-urile
```

### Comparație practică: Counter component

```typescript
// Angular (Classic - Default CD)
@Component({
  template: `
    <div>
      <p>Count: {{ count }}</p>
      <button (click)="increment()">+</button>
    </div>
  `
})
export class CounterComponent {
  count = 0
  increment() { this.count++ }
  // La click:
  // 1. Zone.js detectează click event
  // 2. Angular pornește CD de la root
  // 3. Verifică TOATE componentele din tree
  // 4. La acest component, verifică binding-ul {{ count }}
  // 5. Update DOM
}

// Angular (Signals - Zoneless)
@Component({
  template: `
    <div>
      <p>Count: {{ count() }}</p>
      <button (click)="increment()">+</button>
    </div>
  `
})
export class CounterComponent {
  count = signal(0)
  increment() { this.count.update(v => v + 1) }
  // La click:
  // 1. Signal se schimbă
  // 2. Doar acest component este notificat
  // 3. Doar binding-ul {{ count() }} este verificat
  // 4. Update DOM
}
```

```vue
<!-- Vue 3 (VDOM) -->
<script setup lang="ts">
import { ref } from 'vue'
const count = ref(0)
function increment() { count.value++ }
// La click:
// 1. Proxy setter pe count triggerează notification
// 2. Doar acest component (care citește count) este re-renderat
// 3. Render function generează nou VNode tree
// 4. Patch algorithm (cu patch flags) actualizează doar text node-ul
// 5. Update DOM
</script>

<template>
  <div>
    <p>Count: {{ count }}</p>
    <button @click="increment">+</button>
  </div>
</template>
```

```vue
<!-- Vue 3 (Vapor Mode) -->
<script setup vapor lang="ts">
import { ref } from 'vue'
const count = ref(0)
function increment() { count.value++ }
// La click:
// 1. Proxy setter pe count triggerează notification
// 2. watchEffect-ul compilat actualizează DIRECT text node-ul
// 3. Update DOM (un singur textContent = ... )
// FĂRĂ VNode creation, FĂRĂ diffing
</script>

<template>
  <div>
    <p>Count: {{ count }}</p>
    <button @click="increment">+</button>
  </div>
</template>
```

### Scenarii de performanță comparate

```
Scenariu 1: Aplicație cu 500 componente, 1 input field se schimbă
─────────────────────────────────────────────────────────────────
Angular Default CD:  Verifică ~500 componente (tree walk complet)
Angular OnPush:      Verifică ~1-5 componente (doar affected subtree)
Angular Signals:     Verifică ~1 component (doar cel cu signal)
Vue 3 VDOM:          Re-renderează ~1 component (proxy tracking)
Vue 3 Vapor:         Update direct ~1 DOM node (fără re-render)

Scenariu 2: Lista cu 10,000 items, selecție schimbă pe 1 item
─────────────────────────────────────────────────────────────────
Angular Default CD:  Verifică toate 10,000 binding-uri
Angular OnPush:      Skip dacă @Input ref nu s-a schimbat
Angular Signals:     Update doar item-ul afectat
Vue 3 (fără v-memo): Re-render list component (dar diff e rapid cu keys)
Vue 3 (cu v-memo):   Skip 9,999 items, re-render doar 1-2 items
Vue 3 Vapor:         Update direct binding-urile afectate
```

### Concluzie pentru interviu

> Ambele framework-uri au ajuns la performanțe excelente prin abordări diferite.
> Angular a plecat de la "check everything" (Zone.js + CD) și evoluează spre
> "check only what changed" (Signals). Vue a plecat de la "track dependencies
> automatically" (Proxy reactivity) și evoluează spre "eliminate overhead layers"
> (Vapor Mode). **Direcția finală este aceeași: fine-grained reactivity cu update
> direct al DOM-ului, fără intermediari.**

---

## 13. Întrebări de interviu

### Întrebarea 1: Cum funcționează Virtual DOM în Vue 3? Ce optimizări face compiler-ul?

**Răspuns:**

Virtual DOM în Vue 3 este un arbore de obiecte JavaScript (VNode-uri) care
reflectă structura DOM-ului real. Când starea se schimbă, Vue generează un nou
VNode tree, îl compară cu cel anterior prin algoritmul de patch, și aplică doar
diferențele pe DOM-ul real.

Ce face Vue 3 special comparativ cu alte implementări VDOM (React, de exemplu)
este conceptul de **compiler-informed Virtual DOM**. Compiler-ul Vue analizează
template-ul la build time și adaugă metadata pe VNode-uri: **patch flags** care
indică exact ce se poate schimba (text, class, style, props), **static hoisting**
care extrage nodurile statice în afara funcției render, și **block tree** care
creează un array plat de noduri dinamice, eliminând nevoia de diff recursiv pe
subtree-uri statice. Rezultatul: runtime-ul face O(n) operații unde n e numărul
de noduri **dinamice**, nu totalul de noduri din template.

---

### Întrebarea 2: Cum optimizezi o listă cu 10,000+ items în Vue?

**Răspuns:**

Abordarea pe niveluri, de la cel mai impactant la cel mai puțin:

1. **Virtual scrolling** - cel mai important. Folosesc `vue-virtual-scroller`
   (RecycleScroller pentru items cu aceeași dimensiune, DynamicScroller pentru
   dimensiuni variabile). Renderez doar items-urile vizibile în viewport plus
   un buffer. DOM-ul real conține 20-50 de elemente, nu 10,000.

2. **v-memo** - pe fiecare item din listă, specific exact ce proprietăți
   contează pentru re-render. De exemplu `v-memo="[item.id === selectedId]"`
   face ca la schimbarea selecției doar 2 items să se re-rendereze (cel vechi
   și cel nou selectat), nu toate 10,000.

3. **shallowRef** pentru lista principală - nu am nevoie ca Vue să facă deep
   tracking pe 10,000 de obiecte nested. Schimb referința la update.

4. **:key stabil** - întotdeauna `item.id`, niciodată index.

5. **Paginare server-side** - dacă datele vin de la API, nu aduc toate 10,000
   simultan. Pagination sau cursor-based infinite scroll.

---

### Întrebarea 3: Ce este Vapor Mode și de ce este important?

**Răspuns:**

Vapor Mode este noua strategie de compilare din Vue 3.6+ care elimină complet
Virtual DOM-ul. În loc să genereze funcții render care creează VNode-uri și apoi
fac diff/patch, compiler-ul Vapor generează cod imperativ care manipulează
DOM-ul direct.

E important pentru trei motive. Primo, **performanță** - elimină overhead-ul
de VNode creation și diffing, obținând 2-3x speedup pe updates frecvente.
Secundo, **bundle size** - nu include runtime-ul VDOM, reducând dimensiunea.
Tertio, **convergența industriei** - Solid.js, Svelte, și acum Vue (plus
Angular cu Signals) se îndreaptă toate spre același model: reactivity
fine-grained cu DOM updates directe. Este opt-in per component (keyword
`vapor` în `<script setup>`), backwards compatible, și folosește exact
aceeași Composition API. Nu trebuie să schimbi nimic la nivel de template
sau logică - doar compiler-ul generează cod diferit.

---

### Întrebarea 4: Cum faci code splitting eficient într-o aplicație Vue enterprise?

**Răspuns:**

Code splitting pe mai multe niveluri. La nivel de **rute** - fiecare rută
principală este un lazy-loaded chunk prin dynamic import în router config.
La nivel de **features** - în `vite.config.ts` configurez `manualChunks`
pentru a grupa fișierele pe feature folders (admin, analytics, etc.).
La nivel de **componente** - `defineAsyncComponent` pentru componente grele
care nu sunt necesare la initial load (editori, chart-uri, modals complexe).
La nivel de **vendor** - separ dependințele third-party în chunks separate
pentru long-term caching (vue-core, ui-library, charts, utils).

Strategii de prefetch: prefetch on hover pentru rute probabile, prefetch
after idle pentru rute secundare, și Webpack magic comments pentru
prefetch hints. Monitorizez cu `rollup-plugin-visualizer` pentru a identifica
chunks anormal de mari și dependințe duplicate. Target: initial bundle sub
200KB gzipped, rute secundare sub 50KB fiecare.

---

### Întrebarea 5: v-memo vs v-once - când le folosești?

**Răspuns:**

**v-once** - render o singură dată, niciodată update. Folosesc pentru conținut
care nu se schimbă niciodată: copyright notices, header-uri statice, versiunea
aplicației, legal text. Este drastic - dacă datele se schimbă, utilizatorul
nu va vedea schimbarea.

**v-memo** - memoizare condiționată. Re-renderul se face doar când dependențele
specificate se schimbă. Folosesc pentru liste mari unde fiecare item e costisitor
de renderat dar nu toate proprietățile item-ului afectează vizual render-ul.
Exemplu clasic: tabel cu selecție, `v-memo="[item.id === selectedId]"` -
nu re-renderez un item doar pentru că alt item s-a selectat.

Regula: v-once = date complet statice, v-memo = date dinamice dar cu
invalidation control explicit. v-memo cu array gol este echivalent cu v-once.

---

### Întrebarea 6: Cum diagnostichezi probleme de performanță în Vue?

**Răspuns:**

Procesul meu de diagnoză are mai mulți pași. Primo, **măsor** - Lighthouse
pentru metrici globale (LCP, INP, CLS), Chrome DevTools Performance tab
pentru profiling detaliat, Vue DevTools Performance tab pentru timing per
component (cu `app.config.performance = true`).

Secundo, **identific bottleneck-ul** - dacă LCP e mare, e problem de bundle
size sau SSR. Dacă INP e mare, e re-render overhead. Dacă CLS e mare, e
layout instabilitate. Bundle analyzer (vite-bundle-visualizer) pentru a
identifica dependințe mari sau duplicate.

Tertio, **aplic soluții targeted** - nu optimizez prematur. Dacă un component
e slow, verific de ce: re-renderuri inutile (adaug v-memo), liste mari
(virtual scrolling), obiecte deep reactive inutile (shallowRef), computed-uri
care ar trebui cached (computed vs method), API calls duplicate (caching).

Quarto, **monitorizez continuu** - Web Vitals reporting în producție,
performance budgets în CI/CD (Lighthouse CI), alerting pe regresii.

---

### Întrebarea 7: Cum optimizezi performanța în contextul Micro-Frontend?

**Răspuns:**

MFE adaugă provocări specifice de performanță. **Shared dependencies** -
prin Module Federation, Vue, Vue Router, și Pinia sunt shared între host și
remotes. Asta previne duplicarea (de exemplu, 3 MFE-uri nu încarcă 3 copii
de Vue). Config în Webpack: `shared: { vue: { singleton: true } }`.

**Lazy loading MFE-urilor** - fiecare remote se încarcă doar când ruta aferentă
e accesată. Container-ul host are doar un shell minimal. **Prefetch inteligent**
- pe hover pe navigație, pre-încărc remoteEntry.js al MFE-ului respectiv.

**CSS isolation** - scoped styles sau CSS modules per MFE, pentru a preveni
conflicte și style recalculations. **Shared state minimal** - comunicare
inter-MFE prin event bus sau shared Pinia store, nu prin props drilling
peste boundaries.

**Bundle budgets per MFE** - fiecare remote are budget independent (ex: 150KB
gzipped). Monitorizez cu CI checks. **Fallback UI** - dacă un remote e slow
sau fail, host-ul arată un skeleton/fallback, nu blochează toată aplicația.

---

### Întrebarea 8: KeepAlive - beneficii și riscuri?

**Răspuns:**

**Beneficii:** Păstrează starea componentelor (scroll position, form inputs,
data loaded) la navigare. Utilizatorul revine exact unde a plecat. Elimină
re-fetch-uri și re-renderuri inutile. UX mult mai fluid, similar unei
aplicații native.

**Riscuri și mitigări:** Memory leaks - folosesc `:max` pentru LRU eviction
(de obicei 5-10 componente max). Date stale - în `onActivated` verific
freshness-ul datelor și re-fetch dacă sunt mai vechi de N minute. Event
listeners rămân active - în `onDeactivated` opresc polling-ul, WebSocket
subscriptions, timers. Stare inconsistentă - dacă o entitate a fost
ștearsă pe backend dar componenta e cached cu date vechi, trebuie handled
în `onActivated`.

Folosesc KeepAlive selectiv: `:include` doar pe componente care beneficiază
(liste cu scroll, dashboards cu date). Nu cache-ez formulare (risc de
submitted data stale) sau componente care trebuie mereu fresh.

---

### Întrebarea 9: Cum compari Angular Change Detection cu Vue Reactivity din punct de vedere performanță?

**Răspuns:**

Sunt două filozofii fundamental diferite. Angular (classic) folosește o
abordare **top-down pull**: Zone.js interceptează events, Angular parcurge
tot component tree-ul și verifică fiecare binding. Optimizarea e manuală:
OnPush, detach, markForCheck. Vue folosește o abordare **bottom-up push**:
Proxy-urile notifică automat doar componentele care depind de datele schimbate.
Optimizarea e automată by default.

În practică, o aplicație Vue default performează comparabil cu o aplicație
Angular optimizată cu OnPush peste tot. Angular neoptimizat (default CD
pe 500+ componente) poate fi semnificativ mai lent.

Direcția modernă convergează: Angular Signals elimină Zone.js și adoptă
fine-grained reactivity (similar Vue), iar Vue Vapor Mode elimină VDOM
pentru direct DOM updates. Ambele ajung la aceeași concluzie: reactive
graph cu granularitate la nivel de binding, nu de component.

---

### Întrebarea 10: shallowRef - când și de ce?

**Răspuns:**

`shallowRef` face reactive doar assignment-ul pe `.value`, nu nested
properties. Folosesc în trei scenarii principale:

1. **Liste mari** (1000+ obiecte cu nested data) - deep reactivity creează
   un Proxy pe fiecare obiect și proprietate nested. La 10,000 useri cu
   fiecare having address, permissions, metadata - vorbim de sute de mii
   de Proxy-uri. Cu shallowRef, am un singur Proxy pe array reference.
   Update: `users.value = [...users.value]` sau `triggerRef(users)`.

2. **Instanțe third-party** - Chart.js instances, Map objects, WebSocket
   connections. Aceste obiecte au internal state complex pe care Vue nu
   trebuie să-l observe. Deep reactive pe un Chart.js instance poate cauza
   bugs și overhead.

3. **Cache / read-mostly data** - date care vin de la API, se citesc des
   dar se schimbă rar (configurări, lookup tables). Nu am nevoie de
   reactivity pe fiecare nested field, doar pe întreaga referință.

Trade-off: pierd convenience-ul de `users.value[0].name = 'new'`
triggering update. Câștig performanță semnificativă pe dataset-uri mari.

---

### Întrebarea 11: Cum combini Vapor Mode cu componente VDOM existente?

**Răspuns:**

Vapor Mode este opt-in per component, ceea ce înseamnă că poți mixa liber
componente Vapor și VDOM în aceeași aplicație. Adaugi `vapor` în tag-ul
`<script setup vapor>` pe componentele care beneficiază (cele cu update-uri
frecvente sau cele performance-critical).

Strategia de migrare: nu migrez totul deodată. Identific componentele
cu cel mai mare impact de performanță (liste, real-time dashboards,
componente cu animații/tranziții de date) și le convertesc la Vapor.
Componentele care folosesc KeepAlive, Transition, sau render functions
(h()) rămân pe VDOM deocamdată, până când suportul Vapor se extinde.

Interoperabilitatea funcționează transparent - un component VDOM poate
conține un copil Vapor și invers. Props, emits, provide/inject,
lifecycle hooks - totul funcționează identic. Singura diferență este
sub capotă: modul în care compiler-ul generează codul de update.

---

### Întrebarea 12: Descrie strategia ta completă de performanță pentru o aplicație Vue enterprise cu MFE.

**Răspuns:**

Structurez pe trei axe: **build time**, **load time**, **runtime**.

**Build time:** Code splitting agresiv - fiecare MFE este un chunk separat,
shared dependencies via Module Federation singleton, vendor chunks separate
per category (ui, charts, utils). Manual chunks în Vite config pentru control
precis. Tree-shaking verificat cu bundle analyzer. CSS code splitting activat.
Performance budgets: initial bundle < 200KB gzipped, rute lazy < 50KB.

**Load time:** Lazy loading pe toate rutele. Prefetch on hover pentru
navigație probabilă. Critical CSS inline. Preconnect/preload pentru API
endpoints. Service Worker pentru caching assets. Compression Brotli pe CDN.
Font loading optimizat (font-display: swap, subset). Image optimization
(WebP/AVIF, lazy loading, srcset responsive).

**Runtime:** v-memo pe liste cu 100+ items. shallowRef pentru dataset-uri
mari. Virtual scrolling peste 1000 items. KeepAlive cu max pe views
frecvent navigate. computed (nu methods) pentru derivări. defineAsyncComponent
pentru componente grele off-screen. Debounce pe input events. Web Workers
pentru procesări heavy (parsing CSV, calcule). Considerare Vapor Mode
pentru componente real-time.

**Monitoring:** Web Vitals în producție, Lighthouse CI în pipeline, bundle
size checks pe PR, alerting pe regression.


---

**Următor :** [**08 - TypeScript in Vue** →](Vue/08-TypeScript-in-Vue.md)