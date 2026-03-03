# Întrebări de Interviu - Senior Frontend Architect (Interview Prep)

> 27 de întrebări cu răspunsuri complete la nivel architect.
> Framework răspuns: Context → Options → Trade-offs → Recommendation.
> Focus pe decizii arhitecturale, trade-offs, MFE decisions.
> Include și întrebări comportamentale (leadership, mentoring).

---

## Cuprins

1. [Framework pentru Răspunsuri](#1-framework-pentru-răspunsuri)
2. [Întrebări Arhitectură MFE (8)](#2-întrebări-arhitectură-mfe)
3. [Întrebări Vue.js Tehnice (6)](#3-întrebări-vuejs-tehnice)
4. [Întrebări Design & Patterns (5)](#4-întrebări-design--patterns)
5. [Întrebări DevOps & Deployment (3)](#5-întrebări-devops--deployment)
6. [Întrebări Comportamentale / Leadership (5)](#6-întrebări-comportamentale--leadership)
7. [Întrebări pe care TU le pui la interviu](#7-întrebări-pe-care-tu-le-pui-la-interviu)

---

## 1. Framework pentru Răspunsuri

### Structura CORT

Pentru ORICE întrebare de architect, folosește structura:

1. **Context** - Reformulează problema. Arată că înțelegi scope-ul.
2. **Options** - Prezintă 2-3 opțiuni posibile.
3. **Reasoning** - Explică de ce alegi opțiunea preferată.
4. **Trade-offs** - Ce sacrifici. Ce riscuri rămân. Ce ai face diferit dacă X.

### Exemplu aplicat

**Întrebare:** "Cum ai decide dacă proiectul are nevoie de micro-frontenduri?"

> **Context:** "Decizia de a adopta micro-frontenduri nu e o decizie tehnică, ci una organizațională. Depinde de dimensiunea echipei, frecvența deployment-urilor, și gradul de independență necesar între domenii."
>
> **Options:** "Avem trei abordări principale: (1) Monolith - o aplicație, simplu, dar nu scalează pe echipe mari. (2) Monorepo cu Nx/Turborepo - modularitate cu build comun, bun pentru 2-5 echipe. (3) Micro-frontenduri cu Module Federation - aplicații independente, deployment independent."
>
> **Reasoning:** "Aș evalua: câte echipe sunt (3+ echipe = MFE posibil), dacă deployment independent e o cerință reală, dacă domeniile sunt suficient de decuplate. Dacă răspunsul e da la toate, MFE."
>
> **Trade-offs:** "MFE-urile adaugă complexitate operațională: CI/CD per MFE, shared dependency management, UX consistency. Dacă echipa nu are DevOps maturity, costul e prea mare."

### Sfaturi generale

- **Nu răspunde cu "depinde" fără a elabora** - "depinde de X, Y, Z, și recomandarea mea bazată pe experiență e..."
- **Folosește numere** - "am redus build time de la 12 min la 3 min", "aplicația avea 15 MFE-uri"
- **Admite ce nu știi** - "nu am experiență directă cu X, dar bazat pe similaritatea cu Y..."
- **Oferă recomandare** - nu doar opțiuni, ci "dacă aș fi în situația voastră, aș alege X"
- **Referă Angular ca experiență relevantă** - "am implementat asta în Angular cu Module Federation, și conceptul e identic în Vue"

---

## 2. Întrebări Arhitectură MFE

### Q1: "Cum ai decide dacă proiectul are nevoie de micro-frontenduri?"

**Răspuns:**

Decizia MFE nu e tehnică, e organizațională. Evaluez pe trei axe:

**1. Team topology:** Dacă avem 3+ echipe frontend care lucrează pe domenii diferite (products, checkout, dashboard), MFE face sens. Sub 3 echipe, overhead-ul nu se justifică.

**2. Deployment independence:** Dacă Team A trebuie să aștepte Team B să fie gata înainte de deploy, avem o problemă. MFE rezolvă asta - fiecare echipă deployează independent.

**3. Domain boundaries:** Dacă domeniile sunt clar decuplate (products nu depinde de checkout), MFE e natural. Dacă sunt strâns cuplate, MFE introduce complexitate artificială.

**Red flags (NU alege MFE):** echipă mică (<5 devs), MVP, toate echipele fac deploy împreună, domenii strâns cuplate, lipsă DevOps maturity.

**Green flags (alege MFE):** 3+ echipe autonome, deployment independent necesar, domenii clare, CI/CD matur.

**Din experiența mea:** Am evaluat MFE pe un proiect Angular cu 4 echipe. Am ales MFE pentru că echipele aveau release cycles diferite (o echipă la 2 săptămâni, alta zilnic), și coordonarea deployment-ului devenise bottleneck. Conceptul e identic în Vue - e o decizie organizațională, nu de framework.

---

### Q2: "Explică Module Federation - cum funcționează la nivel tehnic?"

**Răspuns:**

Module Federation (Webpack 5) permite aplicațiilor separate să partajeze cod la runtime, nu la build time.

**Concepte cheie:**
- **Host** = aplicația principală care consumă remote modules
- **Remote** = aplicație care expune module (componente, composables)
- **remoteEntry.js** = manifestul unui remote - container API cu `init()` și `get()`
- **Shared** = dependențe partajate (vue, pinia) - negociate la runtime

**Flow la runtime:**
1. Browser încarcă Host app (main.ts → bootstrap.ts - async boundary obligatoriu)
2. La navigare pe o rută MFE, Host face `import('mfeProducts/ProductList')`
3. Webpack runtime fetch-uiește `remoteEntry.js` de la Remote URL
4. Remote container execută `init(shareScope)` - negociază shared deps
5. Exemplu negociere: "am nevoie de vue@^3.4, host are vue@3.5 → match, folosesc versiunea host-ului"
6. Container execută `get('./ProductList')` - returnează module factory
7. Componenta e disponibilă în Host și se renderizează

**Cel mai important:** `singleton: true` pentru Vue - fără asta, fiecare MFE are propria instanță Vue → reactivitatea se strică, provide/inject nu funcționează.

**Paralela Angular:** Flow-ul e identic. Diferența: Angular folosește `@angular-architects/module-federation` ca wrapper, Vue configurează `ModuleFederationPlugin` direct. Conceptul e 100% același.

---

### Q3: "Cum gestionezi shared state între MFE-uri?"

**Răspuns:**

**Anti-pattern:** Shared Pinia store între MFE-uri. Creează tight coupling - dacă un MFE updatează store-ul, toate se re-renderizează. Forțează alinierea versiunilor Pinia.

**Abordarea mea, în ordinea preferinței:**

1. **Custom Events** (cel mai decuplat):
```typescript
window.dispatchEvent(new CustomEvent('user:login', { detail: { userId: '123', name: 'Emanuel' } }))
```
Low coupling, funcționează cross-framework, ideal pentru events care nu sunt frecvente.

2. **URL params** - pentru navigation state (filter=electronics, page=2). Bookmarkable, zero coupling.

3. **localStorage/sessionStorage** - pentru state persistent (auth token, user preferences). Cu StorageEvent pentru sincronizare.

4. **Shared Pinia** (doar dacă TOATE MFE-urile sunt Vue, aceeași versiune) - configurez pinia ca singleton. Risc: tight coupling, version alignment obligatoriu.

**Rule of thumb:** Minimizează shared state. Fiecare MFE ar trebui să fie cât mai self-contained. Dacă trebuie să partajezi mult state, probabil MFE-urile nu au boundaries corecte.

**Implementare practică:** Am creat un `TypedEventBus` cu tipuri TypeScript stricte pentru toate event-urile cross-MFE. Type safety fără coupling.

---

### Q4: "Ce faci când un remote MFE e indisponibil?"

**Răspuns:**

Resilience într-o arhitectură MFE e critică. Abordarea mea pe trei niveluri:

**Nivel 1 - Loading fallbacks:**
```typescript
const ProductList = defineAsyncComponent({
  loader: () => import('mfeProducts/ProductList'),
  loadingComponent: LoadingSpinner,
  errorComponent: ErrorFallback,
  delay: 200,       // ms înainte de loading indicator
  timeout: 15000,   // ms înainte de error
})
```

**Nivel 2 - Retry logic cu exponential backoff:**
Dacă fetch `remoteEntry.js` eșuează, retry de 3 ori cu delay crescând: 1s → 2s → 4s. Implementat ca wrapper peste `defineAsyncComponent` loader.

**Nivel 3 - Circuit breaker:**
Dacă un MFE eșuează de 3 ori în 5 minute, "deschid circuitul" - nu mai încerc să-l încarc, arăt direct fallback UI. Verific periodic dacă revine (half-open state). Pattern clasic din microservices, aplicat la frontend.

**Fallback UI options:**
- Static message: "Modulul e temporar indisponibil"
- Cached version (Service Worker)
- Feature degradation (pagina funcționează fără MFE-ul respectiv)
- Redirect la MFE standalone URL

**Monitoring:** Alert automat când un MFE e down mai mult de 2 minute. Metrici: load time per MFE, error rate, availability.

---

### Q5: "Cum asiguri consistența UI între MFE-uri?"

**Răspuns:**

Cea mai mare provocare practică în MFE. Abordarea mea pe 5 niveluri:

**1. Design Tokens** - variabile CSS custom properties pentru culori, spacing, typography, border-radius. Distribuite ca package NPM. Toate MFE-urile le importă. Schimbi un token → schimbare globală.

**2. Shared Component Library** - package NPM (`@company/ui-kit`) cu componente base: Button, Input, Modal, DataTable, Form elements. Versioned semantic. Fiecare MFE importă din library.

**3. Monorepo approach** - UI kit-ul trăiește în monorepo alături de MFE-uri. Nx/Turborepo pentru build caching. Schimbare în UI kit → rebuild doar MFE-urile afectate.

**4. Visual Regression Testing** - Chromatic/Percy pe fiecare PR. Detectează diferențe vizuale automat. Previne drift-ul vizual între MFE-uri.

**5. Lint Rules** - ESLint custom rules care enforces folosirea componentelor din design system. Flag când cineva scrie `<button>` în loc de `<BaseButton>`.

**Trade-off:** Design system-ul e o dependență shared. Breaking change afectează TOATE MFE-urile. Soluție: semantic versioning strict, changelogs automatice, deprecation warnings cu minim 2 releases înainte de removal.

---

### Q6: "Cum ai migra de la monolith la MFE?"

**Răspuns:**

**Strangler Fig pattern** - migrezi gradual, nu big bang:

**Faza 1 - Analiză (2-4 săptămâni):**
- Mapez domeniile aplicației (DDD bounded contexts)
- Identific modulul cel mai independent (cel mai puțin coupling cu restul)
- Audit shared state și dependențe între module
- Setup Module Federation pe monolith (devine Host)

**Faza 2 - Primul MFE (4-6 săptămâni):**
- Extrag modulul cel mai independent ca Remote
- Dual running: monolith + MFE coexistă
- Monolith-ul încarcă MFE-ul via Module Federation
- Feature flag: switch între versiunea monolith și MFE
- Totul funcționează ca înainte din perspectiva user-ului

**Faza 3 - Migrare iterativă (ongoing):**
- Extrag următorul MFE (în ordinea independenței)
- Repet pattern-ul: extract → test → switch → cleanup
- Cu fiecare MFE extras, monolith-ul se micșorează

**Faza 4 - Clean up:**
- Host-ul e doar shell (navbar, footer, routing, auth)
- Toate features sunt MFE-uri independente

**Timeline realistă:** 6-12 luni pentru o aplicație medie. Nu grăbi - fiecare MFE extras trebuie testat extensiv.

**Cel mai greu:** Refactorizarea shared state. Dacă monolith-ul are un mega-store (Vuex/NgRx) accesat de toate modulele, trebuie separat ÎNAINTE de extragerea MFE-urilor.

---

### Q7: "Cum gestionezi authentication/authorization în MFE?"

**Răspuns:**

**Principiu: Auth în Host (single source of truth).**

**Authentication flow:**
1. Host-ul gestionează login page, token management, refresh
2. Token-ul e stocat în HttpOnly cookie (securizat - BFF pattern) SAU în memory (auth store)
3. Remote MFE-urile primesc auth state prin:
   - **Custom Event**: host emite `auth:state-changed` cu user info (recomandat)
   - **Shared Pinia store**: dacă toate sunt Vue și e simplu
   - **provide/inject**: dacă remote e child component al host-ului

**Authorization:**
- **Route-level:** Host-ul verifică permissions ÎNAINTE de a încărca MFE-ul (navigation guard)
- **Feature-level:** MFE-ul primește user roles și ascunde/arată funcționalități intern

```typescript
// Host - global navigation guard
router.beforeEach(async (to) => {
  const authStore = useAuthStore()
  if (to.meta.requiresAuth && !authStore.isAuthenticated) {
    return { path: '/login', query: { redirect: to.fullPath } }
  }
  if (to.meta.roles && !to.meta.roles.some(r => authStore.hasRole(r))) {
    return '/403'
  }
})
```

**Token refresh:** Gestionat exclusiv de Host. Dacă token-ul expiră, Host redirect la login. MFE-urile NU gestionează refresh - primesc auth state de la Host. Simplifică enorm logica din remote-uri.

---

### Q8: "Cum faci deployment independent per MFE?"

**Răspuns:**

**Infrastructure:**
- Fiecare MFE are propriul CI/CD pipeline (Azure DevOps)
- Path-based triggers: doar `mfe-products/**` changes → doar pipeline-ul Products
- Build output: static files (JS, CSS, assets) uploadate pe CDN/Azure Storage separat per MFE
- `remoteEntry.js` la URL fix (ex: `https://mfe-products.cdn.example.com/remoteEntry.js`)

**Caching strategy (CRITIC):**
- `remoteEntry.js` → **NEVER cached** (`Cache-Control: no-cache, must-revalidate`) - se schimbă la fiecare deploy
- `main.[hash].js`, `chunk.[hash].js` → **Immutable cache** (`max-age=31536000, immutable`) - hash se schimbă la content change
- `index.html` → **No cache** (entry point Host)

**Versioning:**
- Assets cu content hash (Webpack output: `[name].[contenthash].js`)
- `remoteEntry.js` la URL fix, fără hash - conținutul se actualizează automat
- Nu e nevoie de versioning explicit între MFE-uri (sunt decuplate)

**Rollback (30 secunde):**
- Overwrite `remoteEntry.js` pe CDN cu versiunea anterioară
- CDN cache purge pe `/remoteEntry.js`
- User-ul primește automat versiunea anterioară la next page load
- Alternative: blue-green deployment, feature flags per MFE

**Testing pre-deploy:**
1. Unit + Component tests (pipeline per MFE)
2. Integration test: Host + toate Remote-urile pe staging environment
3. Smoke test post-deploy: verifică `remoteEntry.js` returns 200

---

## 3. Întrebări Vue.js Tehnice

### Q9: "Composition API vs Options API - de ce preferi Composition API?"

**Răspuns:**

Composition API e preferat în proiecte enterprise din trei motive fundamentale:

**1. Code organization by feature, not by option.** În Options API, logica unui feature (ex: search) e fragmentată între `data()`, `computed`, `methods`, `watch`. Cu Composition API, totul e colocated - toate refs, computeds și funcțiile unui feature sunt împreună. La 500+ linii de componentă, diferența e enormă.

**2. Reuse logic cu composables.** `useSearch()`, `useAuth()`, `usePagination()` - funcții reutilizabile care încapsulează logică reactivă. Options API nu avea un pattern clean de reuse: mixins aveau naming conflicts și unclear source of data, renderless components erau verbose.

**3. TypeScript support nativ.** `ref<T>()`, `computed()` - tipurile se propagă automat. Options API avea limitări semnificative cu `this` typing.

**Trade-off:** Composition API are learning curve mai mare. Pentru echipă nouă pe Vue sau proiecte mici, Options API e mai intuitiv. Dar pentru arhitect level, Composition API e standardul.

**Paralela Angular:** Angular nu are acest duality - totul e class-based cu decorators. Conceptual, trecerea de la Options API la Composition API e similară cu trecerea de la Angular class services la funcții pure + signals.

---

### Q10: "ref() vs reactive() - recomandarea ta?"

**Răspuns:**

**Recomandare: `ref()` pentru tot.** Motive:

1. **Consistență** - un singur API, nu trebuie să gândești "e primitive? → ref, e object? → reactive"
2. **Funcționează cu primitive** - `reactive()` NU funcționează cu string, number, boolean
3. **Reassignment safe** - poți face `user.value = newUser`. Cu reactive() nu poți reasigna root-ul obiectului
4. **Destructurare sigură** - cu ref obții Ref-uri. Cu reactive() pierzi reactivitatea la destructurare (trebuie `toRefs()`)
5. **Explicit** - `.value` e verbose dar face clar unde e reactive state (nu se confundă cu variabile locale)

**Când reactive() e OK:** State object intern unde nu faci destructurare. Ex: `const form = reactive({ name: '', email: '', errors: {} })`. Dar chiar și aici, `ref()` funcționează.

**Paralela Angular:** Angular `signal()` ≈ Vue `ref()` - o singură abstractie pentru orice. Angular nu are echivalent `reactive()`, și asta e un avantaj de simplitate.

---

### Q11: "Cum organizezi state management într-o aplicație Vue mare?"

**Răspuns:**

**Layered approach, de la cel mai simplu la cel mai complex:**

| Layer | Mecanism | Scope | Exemplu |
|-------|----------|-------|---------|
| **Component state** | `ref()` / `reactive()` | Local | isDropdownOpen, searchQuery |
| **Composable state** | `useXxx()` | Reutilizabil | useForm(), usePagination() |
| **Feature store** | Pinia store per feature | Feature/domain | useProductStore, useCartStore |
| **App state** | Pinia store global | Aplicație | useAuthStore, useThemeStore |
| **Cross-cutting** | `provide/inject` | Subtree | Theme, locale, feature flags |

**Rules:**
1. Start cu component state. Elevate la Pinia DOAR când mai multe componente au nevoie
2. Un store per feature/domain, NU un mega-store
3. Stores nu importă componente (unidirectional dependency)
4. Business logic în store actions, nu în componente
5. Composables pentru logică reutilizabilă care NU e shared state

**Paralela Angular:** Similar cu Angular: component state → service → NgRx store. Diferența: Pinia e mult mai simplu decât NgRx (fără actions/reducers/effects boilerplate). Similar cu Angular services pattern.

---

### Q12: "Cum optimizezi performanța unei aplicații Vue 3?"

**Răspuns:**

**Quick wins (impact mare, effort mic):**
1. **Route-level code splitting** - lazy load routes cu `() => import('./View.vue')`
2. **`defineAsyncComponent`** - lazy load componente heavy (charts, editors)
3. **`v-memo`** pe liste mari - skip re-render items neschimbate
4. **`shallowRef`** pentru arrays/objects mari - reactivity doar pe reassignment
5. **Computed caching** - `computed()` în loc de methods pentru valori derivate

**Medium effort:**
6. **Virtual scrolling** (`vue-virtual-scroller`) pentru liste 1000+ items
7. **`KeepAlive`** - cache componente între navigări
8. **Bundle analysis** - `rollup-plugin-visualizer` / `webpack-bundle-analyzer`

**Advanced:**
9. **Vapor Mode** (Vue 3.6+) - compilare fără Virtual DOM, direct DOM updates
10. **Web Workers** - offload heavy computation
11. **Service Worker caching** - static assets și API responses

**Măsurare:** Vue DevTools Performance, Lighthouse, Web Vitals (LCP < 2.5s, INP < 200ms, CLS < 0.1).

**Paralela Angular:** Angular: OnPush + Signals + trackBy. Vue: automatic fine-grained reactivity (nu ai nevoie de OnPush equivalent - Vue e deja optimizat by default). Vapor Mode ≈ Angular zoneless signals.

---

### Q13: "Cum gestionezi formulare complexe în Vue?"

**Răspuns:**

Vue nu are Angular Reactive Forms (FormGroup, FormControl, FormArray). Abordarea mea:

**1. `v-model` direct** - pentru formulare simple
```vue
<input v-model="form.name" />
<input v-model="form.email" />
```

**2. `useForm()` composable** - pentru formulare complexe cu validation:
```typescript
function useForm<T extends Record<string, any>>(initialValues: T, schema: ZodSchema) {
  const data = reactive({ ...initialValues })
  const errors = reactive<Record<string, string>>({})
  const touched = reactive<Record<string, boolean>>({})

  function validate(): boolean {
    const result = schema.safeParse(data)
    if (!result.success) {
      Object.assign(errors, result.error.flatten().fieldErrors)
      return false
    }
    return true
  }

  return { data, errors, touched, validate, reset: () => Object.assign(data, initialValues) }
}
```

**3. Validare cu Zod** - schema-based, type-safe, runtime validation

**4. Librării** - VeeValidate (cel mai popular) sau FormKit (form builder)

**Trade-off vs Angular:** Angular Reactive Forms sunt mai puternice out-of-box (dynamic forms, FormArray, built-in validators, status tracking). Vue necesită mai mult custom code sau o librărie. Dar Vue approach e mai flexibil și mai ușor de testat (funcții pure, nu clase cu DI).

---

### Q14: "Error handling global în Vue?"

**Răspuns:**

**Trei niveluri de error handling:**

**1. Global handler** - catch-all pentru erori negestionate:
```typescript
// main.ts
app.config.errorHandler = (err, instance, info) => {
  console.error('[Vue Global Error]', err)
  sendToMonitoring(err, { component: instance?.$options.name, info })
}
```

**2. Component-level error boundary** - `onErrorCaptured` în wrapper component:
```vue
<!-- ErrorBoundary.vue -->
<script setup lang="ts">
const error = ref<Error | null>(null)
onErrorCaptured((err) => {
  error.value = err instanceof Error ? err : new Error(String(err))
  return false // prevent propagation
})
</script>
<template>
  <slot v-if="!error" />
  <div v-else>
    <p>Something went wrong</p>
    <button @click="error = null">Retry</button>
  </div>
</template>
```

**3. Async error handling** - try/catch în composables și store actions:
```typescript
async function fetchData() {
  try {
    data.value = await api.get('/endpoint')
  } catch (err) {
    error.value = err instanceof Error ? err : new Error(String(err))
    sendToMonitoring(err)
  }
}
```

**Best practice:** Combine toate 3: global handler ca safety net, ErrorBoundary pe secțiuni critice, try/catch explicit pentru async operations. Plus monitoring (Sentry) pentru production.

---

## 4. Întrebări Design & Patterns

### Q15: "Cum structurezi un proiect Vue mare?"

**Răspuns:**

**Feature-based structure** - fiecare feature e un folder autonom:

```
src/
├── features/
│   ├── products/
│   │   ├── components/        # ProductList.vue, ProductCard.vue
│   │   ├── composables/       # useProducts.ts, useProductFilter.ts
│   │   ├── stores/            # useProductStore.ts
│   │   ├── api/               # products.api.ts
│   │   ├── types/             # product.types.ts
│   │   └── index.ts           # Barrel export (public API)
│   ├── auth/
│   │   └── ...
│   └── checkout/
│       └── ...
├── shared/                    # Cross-feature: UI kit, utils, types
├── plugins/                   # Vue plugins
└── router/                    # Root router
```

**Reguli:**
1. Features NU importă din alte features direct → prin shared/ sau events
2. Shared/ conține DOAR cod cu adevărat transversal (UI kit, utils, common types)
3. Fiecare feature e un candidat pentru viitor MFE dacă crește
4. Barrel exports (`index.ts`) definesc public API-ul fiecărei feature

**De ce nu layer-based (components/, services/, stores/ la root):** La 50+ componente, nu mai găsești nimic. Feature-based = totul e colocated, easy to find, easy to extract.

**Paralela Angular:** Identic conceptual cu Angular feature modules / standalone component folders. Diferența: Vue nu are NgModule → folder + barrel export e suficient.

---

### Q16: "Ce design patterns folosești frecvent?"

**Răspuns:**

1. **Composable Pattern** - cel mai important în Vue. Funcții `useXxx()` care încapsulează logică reactivă (useAuth, useApi, usePagination). Echivalent Angular Services, dar fără DI ceremony.

2. **Facade Pattern** - composable care simplifică interacțiunea cu multiple stores/APIs:
```typescript
function useProductFacade() {
  const productStore = useProductStore()
  const cartStore = useCartStore()
  const { data: products, loading } = useApi<Product[]>('/products')

  function addToCart(product: Product) {
    cartStore.addItem(product)
    trackAnalytics('add_to_cart', product.id)
  }

  return { products, loading, addToCart }
}
```

3. **Strategy Pattern** - strategii injectate ca funcții (sorting, validation, formatting)
4. **Observer Pattern** - Event Bus pentru comunicare decuplată (esp. în MFE)
5. **Adapter Pattern** - adaptoare API care normalizează răspunsuri din surse diferite
6. **Container/Presentational** - Smart components (logic) + Dumb components (rendering)

---

### Q17: "Cum asiguri code quality într-o echipă mare?"

**Răspuns:**

**Automated (nu depinde de disciplina individuală):**
1. **TypeScript strict** - `strict: true`, `noAny: true` în tsconfig
2. **ESLint + Prettier** - autoformat la save, pre-commit hooks (Husky + lint-staged)
3. **CI checks** - lint, type-check, tests TREBUIE să treacă înainte de merge
4. **Bundle size budgets** - CI fail dacă bundle > threshold

**Process:**
5. **Code review** - checklist: TypeScript types, tests, accessibility, performance, naming
6. **ADR (Architecture Decision Records)** - documentez decizii majore cu motivație și alternative
7. **Testing pyramid** - unit (composables, stores) → component → e2e

**Cultural:**
8. **Boy scout rule** - lasă codul mai curat decât l-ai găsit (refactor incremental)
9. **Tech talks** - bi-săptămânale, fiecare membru prezintă un topic
10. **Pair programming** pe task-uri complexe

---

### Q18: "Cum gestionezi shared code între multiple aplicații Vue?"

**Răspuns:**

**Monorepo cu packages:**
```
monorepo/
├── apps/
│   ├── host-app/
│   ├── mfe-products/
│   └── mfe-checkout/
├── packages/
│   ├── ui-kit/          # Shared components (Button, Modal, DataTable)
│   ├── utils/           # Shared utilities (formatters, validators)
│   ├── types/           # Shared TypeScript types
│   └── event-bus/       # Shared MFE communication
├── turbo.json           # Turborepo config
└── package.json
```

**Tool:** Turborepo sau Nx pentru:
- Build caching (rebuild doar ce s-a schimbat)
- Dependency graph (știe ordinea de build)
- Task pipeline (lint → type-check → test → build)

**Versioning:** Packages interne cu semantic versioning. Changelogs automatice cu conventional commits. Breaking changes cu deprecation warnings.

**Alternativă fără monorepo:** NPM private packages. Fiecare package e un repo separat, publicat pe private npm registry. Pros: total independent. Cons: slower iteration, version management complex.

---

### Q19: "De ce ai ales X și nu Y?" (meta-question)

**Răspuns framework:**

Această întrebare testează capacitatea de a lua decizii informate. Structura:

1. **Business context** - ce problemă rezolvam? Ce constrângeri aveam? (timeline, team size, existing tech)
2. **Evaluare** - ce opțiuni am considerat? Ce criterii am folosit? (performance, DX, learning curve, community)
3. **Decizie** - ce am ales și de ce? (criteriul decisiv)
4. **Rezultat** - ce metrici au confirmat decizia? (build time, deployment frequency, bug rate)
5. **Ce aș face diferit** - cu hindsight, ce aș schimba? (arată self-awareness)

**Exemplu concret:**
"Am ales Pinia peste NgRx pentru un proiect anterior (tradus în context Vue). Motivul: echipa avea 3 mid-level devs, learning curve NgRx era prea mare. Pinia ne-a permis să fim productivi în ziua 1. Trade-off: am pierdut time-travel debugging și strict immutability. Rezultat: development velocity 40% mai mare în primele 3 luni. Cu hindsight, decizia corectă - complexitatea NgRx nu se justifica pentru proiectul nostru."

---

## 5. Întrebări DevOps & Deployment

### Q20: "Cum structurezi CI/CD pentru MFE-uri?"

**Răspuns:**

**Principii:**
1. **Un pipeline per MFE** - independent, path-based trigger
2. **Template pipeline** - YAML template reutilizat de toate MFE-urile
3. **Stages:** lint → type-check → test → build → deploy staging → smoke test → deploy prod
4. **Integration test stage** - host + toate remotes pe staging (weekly sau pre-release)

**Pipeline structure (Azure DevOps):**
```yaml
trigger:
  paths:
    include: [mfe-products/**]  # Doar acest MFE

stages:
  - Validate (lint, type-check)
  - Test (unit tests, coverage)
  - Build (webpack build)
  - Deploy Staging (upload to staging CDN)
  - Smoke Test (remoteEntry.js returns 200)
  - Deploy Production (upload to prod CDN + CDN purge)
```

**Key decisions:**
- Assets cu content hash → immutable cache (1 year)
- `remoteEntry.js` → no cache (NEVER cached!)
- CDN purge doar pe `remoteEntry.js` la deploy
- Rollback: re-upload previous `remoteEntry.js` (30 secunde)

---

### Q21: "Ce e Azure Bicep și cum îl folosești?"

**Răspuns:**

Azure Bicep = **Infrastructure as Code** specific Azure. Limbaj declarativ, compilat în ARM templates. Mai lizibil și concis decât ARM JSON.

**Folosim pentru:**
- Storage Account per MFE (static website hosting)
- CDN Profile + Endpoints per MFE
- Application Insights (monitoring)
- DNS records (custom domains)

```bicep
// Exemplu: Storage per MFE
resource storage 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: 'mfe${mfeName}${env}'
  location: location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
  properties: {
    staticWebsite: {
      indexDocument: 'index.html'
      error404Document: 'index.html'  // SPA routing
    }
  }
}
```

**Avantaj față de portal click-ops:** Reproducible, version controlled, reviewable. Când adaugi un nou MFE, rulezi Bicep template → infrastructure ready.

---

### Q22: "Cum faci rollback rapid?"

**Răspuns:**

**4 strategii, în ordinea vitezei:**

1. **CDN origin switch** (30 secunde) - blue-green: switch CDN origin de la v2 storage la v1 storage
2. **remoteEntry.js overwrite** (30 secunde) - upload versiunea anterioară pe CDN → users primesc versiunea veche la next load
3. **Feature flags** (instant) - disable feature fără redeploy. Folosim LaunchDarkly sau custom flags. MFE-ul se încarcă dar feature-ul e hidden.
4. **Canary rollback** (1 minut) - dacă aveam canary deploy (10% trafic pe v2), redirecționez 100% la v1

**Post-rollback:** Immediate: monitoring confirms recovery. Next day: root cause analysis + fix + redeploy (nu rollback permanent).

**Key insight:** `remoteEntry.js` no-cache e crucial. Dacă e cached, rollback-ul nu are efect instant - users au versiunea veche cached.

---

## 6. Întrebări Comportamentale / Leadership

### Q23: "Cum faci mentoring pentru junior/mid developers?"

**Răspuns:**

1. **Code review ca instrument de learning** - nu doar approve/reject, ci: explică DE CE o abordare e mai bună, oferă alternative, link-uri la documentație. Review-ul trebuie să fie o conversație, nu un examen.

2. **Pair programming** pe task-uri complexe - demonstrez patterns, apoi las developer-ul să implementeze. "Watch one, do one, teach one."

3. **Tech talks bi-săptămânale** - fiecare membru al echipei prezintă un topic. Beneficiu dublu: cel care prezintă învață cel mai mult pregătind, ceilalți învață ascultând.

4. **Growth plans personalizate** - discuții 1:1 despre ce vrea să învețe, apoi asignez task-uri targetate. Junior vrea să învețe testing? → Următorul ticket e un task de testing.

5. **Safe-to-fail environment** - "E OK să faci greșeli, nu e OK să nu înveți din ele." Review cu blândețe, never public criticism.

6. **Documentation** - ADR-uri, runbooks, onboarding docs. Cunoștința nu trebuie să fie doar în capul meu.

---

### Q24: "Cum gestionezi technical debt?"

**Răspuns:**

1. **Tech debt register** - document viu (Notion/Confluence) cu toate items-urile de tech debt, fiecare cu: descriere, impact, effort estimat, owner.

2. **Matricea Impact vs Effort:**
   - High impact + Low effort → Fă ACUM (boy scout rule)
   - High impact + High effort → Planifică în sprint
   - Low impact + Low effort → Boy scout rule (fix when touching that code)
   - Low impact + High effort → Documentează, revisit quarterly

3. **20% rule** - alocez ~20% din sprint capacity pentru tech debt. Nu e negociabil cu product owner-ul - e "cost of doing business."

4. **Metric tracking** - bundle size trend, build time trend, test coverage trend, TypeScript strictness coverage. Dacă metricile se degradează → tech debt e prioritizat.

5. **Prevention** - code review strict, lint rules, architecture guardrails. Cel mai bun tech debt e cel care nu se creează.

---

### Q25: "Cum iei decizii arhitecturale când echipa nu e de acord?"

**Răspuns:**

1. **RFC (Request for Comments)** - scriu propunerea într-un document structurat (problem, options, recommendation, trade-offs). Echipa comentează async - toți au voce.

2. **Proof of concept** - dacă e controversat, construim POC-uri pentru top 2 opțiuni. Nu decizii bazate pe opinii, ci pe date.

3. **Decision criteria explicite** - definesc upfront CE criterii contează (performance, DX, maintenance cost, learning curve) și ponderea fiecăruia. Evaluăm opțiunile obiectiv.

4. **Disagree and commit** - după dezbatere, se alege o direcție. TOȚI o urmează, chiar dacă nu toți sunt de acord. Sabotajul pasiv e inacceptabil.

5. **ADR (Architecture Decision Record)** - documentez decizia, motivele, alternativele respinse, și CE ar trebui să se întâmple ca să reconsiderăm. Previne re-litigarea aceleiași decizii peste 3 luni.

6. **Reversible vs irreversible** - pentru decizii reversibile, bias spre acțiune ("hai să încercăm"). Pentru irreversibile, bias spre precauție (mai multă analiză, mai mult consens).

---

### Q26: "Descrie o situație în care ai făcut o greșeală tehnică"

**Răspuns (STAR format):**

**Situație:** Pe un proiect anterior, am ales să implementez un shared global store monolitic (echivalent un mega-NgRx store) pentru TOATĂ aplicația, în loc de stores per feature.

**Task:** State management pentru o aplicație cu 6 module/features.

**Acțiune greșită:** Am pus totul într-un singur store "pentru simplitate." Inițial a mers bine.

**Rezultat negativ:** Pe măsură ce proiectul a crescut la 15+ developeri și 40+ entități:
- Store-ul avea 2000+ linii
- Re-render-uri inutile (orice schimbare trigera update-uri în componente care nu aveau nevoie)
- Tight coupling: echipele se blocau reciproc
- Testing devenise un coșmar (mock-uri enorme)

**Rezolvare:** Am refactorizat incremental pe parcursul a 3 luni:
- Split mega-store în stores per feature
- Eliminat dependențele cross-feature
- Introdus event-based communication unde era necesar

**Learning:** State management trebuie gândit per-feature de la început. E mult mai ușor să combini stores mai târziu decât să spargi un mega-store. Acest learning se aplică direct la Pinia - setup stores per feature din ziua 1.

---

### Q27: "Cum comunici decizii tehnice către stakeholders non-tehnici?"

**Răspuns:**

1. **Analogii:** "MFE-urile sunt ca apartamentele într-un bloc - fiecare familie (echipă) are propriul spațiu, dar partajează infrastructura (apă, electricitate = shared deps)."

2. **Focus pe business impact:** Nu "Module Federation reduce bundle size cu 30%", ci "Utilizatorii vor vedea pagina cu 2 secunde mai repede, ceea ce reduce bounce rate-ul cu ~15%."

3. **Diagrame:** Un diagram clar valorează mai mult decât 30 minute de explicație. Folosesc Miro/Excalidraw pentru arhitectură, flow-uri, dependencies.

4. **Evită jargon:** "Sistem de componente partajate" nu "shared federated modules with singleton dependency resolution."

5. **ROI framing:** "Investim 2 sprints acum, economisim 1 zi per deploy pe viitor. La 50 deploy-uri pe an, economia e de 50 zile-om."

6. **Regular updates:** 5 minute la standup, nu 30 minute la quarterly review. Stakeholders preferă frecvent și scurt.

---

## 7. Întrebări pe care TU le pui la interviu

### Despre proiect

- "Câte micro-frontenduri aveți și câte echipe lucrează pe ele?"
- "Ce versiune de Vue folosiți? Ați migrat de la Vue 2?"
- "Cum gestionați shared dependencies? Aveți probleme de version mismatch?"
- "Ce challenges ați avut cu Module Federation?"
- "Cum arată pipeline-ul de deployment? Cât durează un deploy?"
- "Aveți un design system shared? Cum îl distribuiți?"
- "Care e gradul de code coverage? Ce tool-uri de testing folosiți?"

### Despre echipă

- "Cum e structurată echipa? Câți frontend devs per MFE?"
- "Care e procesul de code review?"
- "Cum luați decizii arhitecturale? Aveți ADR-uri?"
- "Ce tool-uri folosiți pentru monitoring și alerting?"
- "Cum e relația frontend-backend? Ownership clear?"

### Despre rol

- "Care ar fi primele 3 luni? Ce așteptări concrete aveți?"
- "Care sunt cele mai mari challenges arhitecturale acum?"
- "Ce grad de autonomie am în decizii tehnice?"
- "Am oameni de mentorat? Ce nivele de experiență?"
- "Care e procesul de promovare și evaluare?"

### De ce aceste întrebări

Aceste întrebări demonstrează:
- **Gândire strategică** - nu întrebi despre syntax, ci despre architecture
- **Experiență** - știi ce întrebări să pui pentru că ai trăit situații similare
- **Proactivitate** - vrei să înțelegi context-ul ÎNAINTE de a face promisiuni
- **Fit cultural** - te interesează echipa și procesul, nu doar tech-ul
- **Maturity** - întrebările despre expectations și challenges arată că ești realist

---

## Checklist pre-interviu

- [ ] Revizuit fișierul [05-Micro-Frontenduri-Module-Federation.md](./05-Micro-Frontenduri-Module-Federation.md)
- [ ] Exersat răspunsurile CORT cu voce tare (nu doar în cap)
- [ ] Pregătit 2-3 "war stories" din experiența Angular (transferabile)
- [ ] Pregătit întrebările pe care le pui TU
- [ ] Revizuit [13-Pitch-Personal-Vue.md](./13-Pitch-Personal-Vue.md) - talking points
- [ ] Testat rapid un Hello World Vue 3 cu Composition API (hands-on familiarity)
