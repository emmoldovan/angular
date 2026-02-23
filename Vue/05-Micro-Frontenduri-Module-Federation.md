# Micro-Frontenduri și Module Federation (Interview Prep - Senior Frontend Architect)

> Ghid exhaustiv pentru arhitectura micro-frontenduri cu Vue 3 și Module Federation (Webpack 5).
> Configurare host/remote, shared dependencies, routing, comunicare inter-MFE,
> error handling, resilience, CI/CD, Native Federation.
> **Acesta este cel mai important topic pentru interviu.**

---

## Cuprins

1. Ce sunt Micro-Frontendurile
2. Decision Framework: Când MFE vs Monolith vs Monorepo
3. Module Federation Webpack 5 - Cum Funcționează
4. Configurare Vue MFE - Host App
5. Configurare Vue MFE - Remote App
6. Shared Dependencies Management
7. Routing în MFE
8. Comunicare Inter-MFE
9. Error Handling și Resilience
10. CI/CD pentru MFE
11. Performanță în MFE
12. Native Federation (@softarc/native-federation)
13. Paralela completă: Angular MFE vs Vue MFE
14. Întrebări de interviu MFE (20+)

---

## 1. Ce sunt Micro-Frontendurile

### Definiție

Micro-frontendurile extind conceptul de microservices la frontend. O aplicație mare e împărțită în aplicații mai mici, independente, care:

- Sunt **dezvoltate** de echipe autonome
- Sunt **testate** independent
- Sunt **deployate** independent (cel mai important beneficiu)
- Pot folosi **tehnologii diferite** (dar de obicei se alege una)
- Comunică prin **contracte bine definite**

### Diagrama Arhitecturală

```
┌──────────────────────────────────────────────────────────────┐
│                     BROWSER                                   │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                   HOST APPLICATION                       │ │
│  │  ┌──────────┐  ┌───────────────┐  ┌────────────────┐   │ │
│  │  │  Navbar   │  │  Remote MFE   │  │  Remote MFE    │   │ │
│  │  │  (Host)   │  │  Products     │  │  Checkout      │   │ │
│  │  │          │  │  (Team A)     │  │  (Team B)      │   │ │
│  │  └──────────┘  └───────────────┘  └────────────────┘   │ │
│  │                                                          │ │
│  │  ┌──────────────────────────────────────────────────┐   │ │
│  │  │              Remote MFE - Dashboard               │   │ │
│  │  │              (Team C)                             │   │ │
│  │  └──────────────────────────────────────────────────┘   │ │
│  └─────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

### Beneficii

| Beneficiu | Descriere | Exemplu Concret |
|-----------|-----------|-----------------|
| **Team Autonomy** | Fiecare echipă deține un MFE end-to-end | Team A lucrează pe Products fără să depindă de Team B |
| **Independent Deployment** | Deploy un MFE fără a afecta restul | Fix urgent în Checkout fără rebuild la Products |
| **Technology Flexibility** | Fiecare MFE poate folosi tech diferit | Products în Vue 3, Legacy Dashboard în Vue 2 |
| **Scalability** | Adaugă echipe fără bottleneck | De la 2 la 10 echipe fără conflicte de merge |
| **Fault Isolation** | Un MFE care cade nu afectează restul | Checkout offline → Products funcționează normal |
| **Incremental Upgrades** | Migrezi gradual, nu big bang | Vue 3.4 → 3.5 per MFE, nu toate odată |

### Trade-offs / Dezavantaje

| Dezavantaj | Impact | Mitigare |
|-----------|--------|---------|
| **Complexitate operațională** | CI/CD per MFE, monitoring, logging | Pipeline templates, centralized monitoring |
| **Bundle size overhead** | Shared deps duplication posibilă | Module Federation shared config |
| **UX consistency** | Design differences între MFE-uri | Shared design system, design tokens |
| **Debugging difficulty** | Bugs cross-MFE greu de reproduss | Distributed tracing, centralized logging |
| **Performance overhead** | Network requests extra, parsing JS | Preloading, caching, lazy loading inteligent |
| **Shared state complexity** | Sincronizare state între MFE-uri | Event-based communication, contracts |

### Tipuri de Integrare MFE

| Tip | Cum funcționează | Pro | Contra | Când |
|-----|-----------------|-----|--------|------|
| **Build-time** (npm packages) | MFE ca npm package, importat la build | Simplu | Deploy dependent, versioning hell | Componente shared |
| **iframes** | Fiecare MFE în iframe | Izolare totală | Performance, UX, comunicare grea | Legacy integration |
| **Runtime JS** (Module Federation) | MFE încărcat dinamic la runtime | Independent deploy, shared deps | Complexitate config | **RECOMANDAT** |
| **Web Components** | MFE ca Custom Elements | Framework agnostic | Shadow DOM limitations, bundle size | Multi-framework |
| **Server-Side** (SSI/ESI) | Compoziție pe server | SEO, performance | Server complexity | Content sites |

---

## 2. Decision Framework: Când MFE vs Monolith vs Monorepo

### Diagrama de Decizie

```
                    Câte echipe lucrează pe frontend?
                              │
                    ┌─────────┼──────────┐
                    ▼         ▼          ▼
                  1-2       3-5         5+
                  echipe    echipe      echipe
                    │         │          │
                    ▼         ▼          ▼
                MONOLITH   EVALUEAZĂ   MICRO-FRONTENDURI
                sau        │            │
                MONOREPO   │            └── Cu Module Federation
                           │
                    ┌──────┼──────┐
                    ▼      ▼      ▼
                  DA      DA      NU
               Domenii  Deploy    │
                clare?  indep?    ▼
                    │      │    MONOREPO
                    ▼      ▼    cu Nx/Turbo
                   MFE    MFE
```

### Matricea de Decizie

| Criteriu | Monolith | Monorepo (Nx/Turbo) | Micro-Frontenduri |
|----------|----------|---------------------|-------------------|
| **Nr. echipe** | 1-2 | 2-5 | 3+ |
| **Deploy frequency** | Same release | Coordinated | Independent |
| **Tech diversity** | Nu | Limitat | Da |
| **Complexitate ops** | Scăzută | Medie | Ridicată |
| **Performance** | Cea mai bună | Bună | Overhead |
| **UX consistency** | Ușoară | Ușoară | Dificilă |
| **Developer experience** | Simplă | Bună | Complexă |
| **Scalare echipe** | Dificilă | Bună | Cea mai bună |
| **Time to market** | Rapid (inițial) | Rapid | Mai lent (inițial), rapid pe termen lung |
| **Cost infrastructure** | Scăzut | Scăzut | Ridicat |

### Anti-patterns: Când NU alegi MFE

1. **Echipă mică** (sub 5 devs total) - overhead-ul MFE nu se justifică
2. **MVP / POC** - pierzi timp pe infrastructure
3. **Aplicație simplă** (CRUD, dashboard) - monolith e suficient
4. **DevOps immaturity** - fără CI/CD solid, MFE e un coșmar
5. **Toate echipele fac deploy împreună** - pierzi beneficiul principal
6. **Tight coupling** între domenii - MFE forțează decoupling, dacă nu se poate, nu forța

### Paralela cu Angular

Decizia MFE vs Monolith e **identică** indiferent de framework. Module Federation e o tehnologie Webpack, nu Angular sau Vue. Diferența e în cum configurezi host/remote (mai simplu în Vue).

---

## 3. Module Federation Webpack 5 - Cum Funcționează

### Concepte Fundamentale

| Concept | Descriere | Exemplu |
|---------|-----------|---------|
| **Container** | Un build webpack care participă în federation | Orice aplicație cu ModuleFederationPlugin |
| **Host** | Aplicația care consumă module remote | Shell app / main app |
| **Remote** | Aplicația care expune module | MFE Products, MFE Checkout |
| **Exposed Module** | Un modul (component, composable) exportat de remote | `./ProductList`, `./useCart` |
| **Shared Module** | Dependență partajată între host și remotes | vue, vue-router, pinia |
| **remoteEntry.js** | Fișierul manifest al unui remote | Container API: init() + get() |

### Cum Funcționează la Runtime

```
1. Browser încarcă HOST app (main.js)
   │
2. Host are bootstrap.ts pattern (async boundary)
   │  └── Necesar pentru shared dependency negotiation
   │
3. Când user-ul navighează la o rută MFE:
   │
4. Host face dynamic import: import('mfeProducts/ProductList')
   │
5. Webpack runtime verifică dacă 'mfeProducts' e disponibil
   │  └── NU → Fetch remoteEntry.js de la Remote URL
   │
6. remoteEntry.js se execută → creează container
   │  ├── container.init(shareScope) → share dependencies
   │  │   ├── Host oferă: vue@3.5.0 (singleton)
   │  │   ├── Remote cere: vue@^3.4.0
   │  │   └── Match! → Remote folosește vue din Host
   │  │
   │  └── container.get('./ProductList') → returnează module factory
   │
7. Module factory se execută → component Vue disponibil
   │
8. Host renderizează componenta remote
```

### ModuleFederationPlugin - Configurare Detaliată

```javascript
const { ModuleFederationPlugin } = require('webpack').container

// REMOTE CONFIG
new ModuleFederationPlugin({
  // Numele unic al acestui container
  name: 'mfeProducts',

  // Fișierul entry point pentru acest remote
  filename: 'remoteEntry.js',

  // Ce module expune acest remote
  exposes: {
    // key = public name, value = internal path
    './ProductList': './src/components/ProductList.vue',
    './ProductDetail': './src/components/ProductDetail.vue',
    './useProducts': './src/composables/useProducts.ts',
    './productRoutes': './src/router/routes.ts',
  },

  // Dependențe partajate
  shared: {
    vue: {
      singleton: true,         // O SINGURĂ instanță (CRITIC pentru Vue!)
      requiredVersion: '^3.4.0', // Semver range acceptat
      strictVersion: false,     // true = throw error dacă nu match
      eager: false,            // true = load imediat (nu lazy)
    },
    'vue-router': {
      singleton: true,
      requiredVersion: '^4.3.0',
    },
    pinia: {
      singleton: true,
      requiredVersion: '^2.1.0',
    },
  },
})

// HOST CONFIG
new ModuleFederationPlugin({
  name: 'hostApp',

  // Remote-urile pe care le consumă acest host
  remotes: {
    // name: 'containerName@URL/remoteEntry.js'
    mfeProducts: 'mfeProducts@http://localhost:3001/remoteEntry.js',
    mfeCheckout: 'mfeCheckout@http://localhost:3002/remoteEntry.js',
    mfeDashboard: 'mfeDashboard@http://localhost:3003/remoteEntry.js',
  },

  shared: {
    vue: { singleton: true, eager: true },      // eager: true în HOST
    'vue-router': { singleton: true, eager: true },
    pinia: { singleton: true, eager: true },
  },
})
```

### Build-time vs Runtime Federation

| Aspect | Build-time Integration | Runtime Integration (MF) |
|--------|----------------------|--------------------------|
| Când se rezolvă | La build (npm install) | La runtime (browser) |
| Versioning | package.json version | remoteEntry.js |
| Deploy independence | NU (trebuie rebuild all) | DA |
| Bundle duplication | Tree-shaked | Shared config |
| Performance | Mai bună (optimized) | Overhead mic |
| Flexibility | Scăzută | Ridicată |
| **Recomandare** | Shared libraries (UI kit) | **MFE-uri** |

---

## 4. Configurare Vue MFE - Host App

### Structura Proiect Host

```
host-app/
├── src/
│   ├── main.ts                # Entry point - import('./bootstrap')
│   ├── bootstrap.ts           # Async bootstrap (CRITICAL!)
│   ├── App.vue               # Root component cu router-view
│   ├── router/
│   │   └── index.ts          # Routes + remote component loading
│   ├── components/
│   │   ├── AppShell.vue      # Layout: header, sidebar, footer
│   │   ├── RemoteWrapper.vue # Generic wrapper for remote components
│   │   ├── LoadingFallback.vue
│   │   └── ErrorFallback.vue
│   ├── composables/
│   │   ├── useRemoteComponent.ts  # Dynamic remote loading
│   │   └── useEventBus.ts        # Cross-MFE communication
│   ├── stores/
│   │   └── useAppStore.ts    # Global app state (auth, theme, etc.)
│   ├── types/
│   │   └── remotes.d.ts      # TypeScript declarations for remotes
│   └── assets/
├── webpack.config.js
├── package.json
└── tsconfig.json
```

### main.ts - Bootstrap Pattern (CRITICAL)

```typescript
// main.ts - TREBUIE să fie async!
// Fără acest pattern, shared dependencies nu sunt disponibile
import('./bootstrap')
```

```typescript
// bootstrap.ts
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import { createRouter, createWebHistory } from 'vue-router'
import App from './App.vue'
import { routes } from './router'

const app = createApp(App)

// Plugins
const pinia = createPinia()
app.use(pinia)

const router = createRouter({
  history: createWebHistory(),
  routes,
})
app.use(router)

// Global error handler
app.config.errorHandler = (err, instance, info) => {
  console.error('[Global Error]', err, info)
  // Send to monitoring service
}

app.mount('#app')
```

**De ce bootstrap pattern?**
Module Federation necesită un async boundary pentru a negocia shared dependencies ÎNAINTE ca aplicația să pornească. Fără `import('./bootstrap')`, vue ar fi loaded eager și fiecare MFE ar avea propria instanță de Vue → broken reactivity.

### Router Setup cu Remote Components

```typescript
// router/index.ts
import { defineAsyncComponent, h } from 'vue'
import type { RouteRecordRaw } from 'vue-router'
import HomeView from '@/views/HomeView.vue'
import ErrorFallback from '@/components/ErrorFallback.vue'
import LoadingFallback from '@/components/LoadingFallback.vue'

// Helper: creează async component cu error/loading handling
function createRemoteComponent(loader: () => Promise<any>) {
  return defineAsyncComponent({
    loader,
    loadingComponent: LoadingFallback,
    errorComponent: ErrorFallback,
    delay: 200,
    timeout: 15000,
  })
}

// Remote components
const ProductList = createRemoteComponent(() => import('mfeProducts/ProductList'))
const ProductDetail = createRemoteComponent(() => import('mfeProducts/ProductDetail'))
const CheckoutPage = createRemoteComponent(() => import('mfeCheckout/CheckoutPage'))
const Dashboard = createRemoteComponent(() => import('mfeDashboard/Dashboard'))

export const routes: RouteRecordRaw[] = [
  {
    path: '/',
    component: HomeView,
  },
  {
    path: '/products',
    component: ProductList,
  },
  {
    path: '/products/:id',
    component: ProductDetail,
    props: true,
  },
  {
    path: '/checkout',
    component: CheckoutPage,
    meta: { requiresAuth: true },
  },
  {
    path: '/dashboard',
    component: Dashboard,
    meta: { requiresAuth: true, roles: ['admin'] },
  },
  {
    path: '/:pathMatch(.*)*',
    component: () => import('@/views/NotFound.vue'),
  },
]
```

### TypeScript Declarations pentru Remote Modules

```typescript
// types/remotes.d.ts
declare module 'mfeProducts/ProductList' {
  import { DefineComponent } from 'vue'
  const component: DefineComponent<{}, {}, any>
  export default component
}

declare module 'mfeProducts/ProductDetail' {
  import { DefineComponent } from 'vue'
  interface Props {
    id: string | number
  }
  const component: DefineComponent<Props, {}, any>
  export default component
}

declare module 'mfeCheckout/CheckoutPage' {
  import { DefineComponent } from 'vue'
  const component: DefineComponent<{}, {}, any>
  export default component
}

declare module 'mfeDashboard/Dashboard' {
  import { DefineComponent } from 'vue'
  const component: DefineComponent<{}, {}, any>
  export default component
}
```

### Webpack Config complet - Host

```javascript
// webpack.config.js - HOST
const path = require('path')
const { VueLoaderPlugin } = require('vue-loader')
const HtmlWebpackPlugin = require('html-webpack-plugin')
const { ModuleFederationPlugin } = require('webpack').container
const { DefinePlugin } = require('webpack')

module.exports = (env, argv) => {
  const isProduction = argv.mode === 'production'

  // Dynamic remote URLs based on environment
  const remoteUrls = {
    mfeProducts: isProduction
      ? 'https://mfe-products.example.com'
      : 'http://localhost:3001',
    mfeCheckout: isProduction
      ? 'https://mfe-checkout.example.com'
      : 'http://localhost:3002',
    mfeDashboard: isProduction
      ? 'https://mfe-dashboard.example.com'
      : 'http://localhost:3003',
  }

  return {
    entry: './src/main.ts',
    mode: argv.mode || 'development',
    output: {
      path: path.resolve(__dirname, 'dist'),
      publicPath: 'auto',
      clean: true,
    },
    resolve: {
      extensions: ['.ts', '.js', '.vue', '.json'],
      alias: {
        '@': path.resolve(__dirname, 'src'),
      },
    },
    module: {
      rules: [
        {
          test: /\.vue$/,
          loader: 'vue-loader',
        },
        {
          test: /\.ts$/,
          loader: 'ts-loader',
          options: { appendTsSuffixTo: [/\.vue$/] },
          exclude: /node_modules/,
        },
        {
          test: /\.css$/,
          use: ['style-loader', 'css-loader'],
        },
      ],
    },
    plugins: [
      new VueLoaderPlugin(),
      new HtmlWebpackPlugin({
        template: './public/index.html',
      }),
      new DefinePlugin({
        __VUE_OPTIONS_API__: false,
        __VUE_PROD_DEVTOOLS__: false,
        __VUE_PROD_HYDRATION_MISMATCH_DETAILS__: false,
      }),
      new ModuleFederationPlugin({
        name: 'hostApp',
        remotes: {
          mfeProducts: `mfeProducts@${remoteUrls.mfeProducts}/remoteEntry.js`,
          mfeCheckout: `mfeCheckout@${remoteUrls.mfeCheckout}/remoteEntry.js`,
          mfeDashboard: `mfeDashboard@${remoteUrls.mfeDashboard}/remoteEntry.js`,
        },
        shared: {
          vue: { singleton: true, eager: true, requiredVersion: '^3.4.0' },
          'vue-router': { singleton: true, eager: true, requiredVersion: '^4.3.0' },
          pinia: { singleton: true, eager: true, requiredVersion: '^2.1.0' },
        },
      }),
    ],
    devServer: {
      port: 3000,
      historyApiFallback: true,
      hot: true,
      headers: {
        'Access-Control-Allow-Origin': '*',
      },
    },
  }
}
```

---

## 5. Configurare Vue MFE - Remote App

### Structura Proiect Remote

```
mfe-products/
├── src/
│   ├── main.ts              # Standalone entry (dev mode)
│   ├── bootstrap.ts         # Async bootstrap
│   ├── App.vue             # Standalone wrapper (dev mode)
│   ├── components/
│   │   ├── ProductList.vue  # ← EXPOSED
│   │   ├── ProductDetail.vue # ← EXPOSED
│   │   └── ProductCard.vue  # Internal (not exposed)
│   ├── composables/
│   │   └── useProducts.ts   # ← EXPOSED
│   ├── stores/
│   │   └── useProductStore.ts
│   ├── router/
│   │   └── routes.ts        # ← EXPOSED (opțional)
│   └── types/
│       └── product.types.ts
├── webpack.config.js
├── package.json
└── tsconfig.json
```

### Dual Mode: Standalone + Remote

Remote-ul trebuie să funcționeze ȘI standalone (pentru development) ȘI ca remote (consumat de host).

```typescript
// main.ts
import('./bootstrap')

// bootstrap.ts
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import { createRouter, createWebHistory } from 'vue-router'
import App from './App.vue'

// Standalone mode - propriul router și mount
const app = createApp(App)
app.use(createPinia())
app.use(createRouter({
  history: createWebHistory(),
  routes: [
    { path: '/', redirect: '/products' },
    { path: '/products', component: () => import('./components/ProductList.vue') },
    { path: '/products/:id', component: () => import('./components/ProductDetail.vue') },
  ],
}))
app.mount('#app')
```

### Componenta Expusă - ProductList.vue

```vue
<!-- components/ProductList.vue -->
<script setup lang="ts">
import { ref, onMounted, computed } from 'vue'
import ProductCard from './ProductCard.vue'
import { useProductStore } from '../stores/useProductStore'
import { storeToRefs } from 'pinia'

const productStore = useProductStore()
const { products, loading, error } = storeToRefs(productStore)
const { fetchProducts, setFilter } = productStore

const searchQuery = ref('')
const filteredProducts = computed(() => {
  if (!searchQuery.value) return products.value
  return products.value.filter(p =>
    p.name.toLowerCase().includes(searchQuery.value.toLowerCase())
  )
})

onMounted(() => {
  fetchProducts()
})
</script>

<template>
  <div class="product-list">
    <div class="search-bar">
      <input
        v-model="searchQuery"
        placeholder="Caută produse..."
        type="search"
      />
    </div>

    <div v-if="loading" class="loading">Se încarcă produsele...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else class="products-grid">
      <ProductCard
        v-for="product in filteredProducts"
        :key="product.id"
        :product="product"
        @add-to-cart="$emit('add-to-cart', $event)"
      />
    </div>
  </div>
</template>
```

### Webpack Config - Remote

```javascript
// webpack.config.js - REMOTE (mfe-products)
const path = require('path')
const { VueLoaderPlugin } = require('vue-loader')
const HtmlWebpackPlugin = require('html-webpack-plugin')
const { ModuleFederationPlugin } = require('webpack').container
const { DefinePlugin } = require('webpack')

module.exports = {
  entry: './src/main.ts',
  mode: 'development',
  output: {
    path: path.resolve(__dirname, 'dist'),
    publicPath: 'auto',
    clean: true,
  },
  resolve: {
    extensions: ['.ts', '.js', '.vue', '.json'],
    alias: {
      '@': path.resolve(__dirname, 'src'),
    },
  },
  module: {
    rules: [
      { test: /\.vue$/, loader: 'vue-loader' },
      {
        test: /\.ts$/,
        loader: 'ts-loader',
        options: { appendTsSuffixTo: [/\.vue$/] },
        exclude: /node_modules/,
      },
      { test: /\.css$/, use: ['style-loader', 'css-loader'] },
    ],
  },
  plugins: [
    new VueLoaderPlugin(),
    new HtmlWebpackPlugin({ template: './public/index.html' }),
    new DefinePlugin({
      __VUE_OPTIONS_API__: false,
      __VUE_PROD_DEVTOOLS__: false,
      __VUE_PROD_HYDRATION_MISMATCH_DETAILS__: false,
    }),
    new ModuleFederationPlugin({
      name: 'mfeProducts',
      filename: 'remoteEntry.js',
      exposes: {
        './ProductList': './src/components/ProductList.vue',
        './ProductDetail': './src/components/ProductDetail.vue',
        './useProducts': './src/composables/useProducts.ts',
      },
      shared: {
        vue: { singleton: true, requiredVersion: '^3.4.0' },
        'vue-router': { singleton: true, requiredVersion: '^4.3.0' },
        pinia: { singleton: true, requiredVersion: '^2.1.0' },
      },
    }),
  ],
  devServer: {
    port: 3001,
    historyApiFallback: true,
    headers: { 'Access-Control-Allow-Origin': '*' },
  },
}
```

---

## 6. Shared Dependencies Management

### De Ce E Critic

Vue **TREBUIE** să fie singleton. Dacă host-ul și remote-ul au instanțe diferite de Vue:
- Reactivitatea nu funcționează cross-MFE
- provide/inject nu funcționează
- Vue DevTools nu detectează componentele

### Configurare Shared Dependencies

```javascript
shared: {
  // OBLIGATORIU singleton
  vue: {
    singleton: true,          // O singură instanță
    requiredVersion: '^3.4.0', // Range acceptat
    strictVersion: false,      // false = warning, true = error
    eager: false,             // true doar în HOST
  },
  'vue-router': {
    singleton: true,
    requiredVersion: '^4.3.0',
  },
  pinia: {
    singleton: true,
    requiredVersion: '^2.1.0',
  },

  // OPȚIONAL shared (NOT singleton)
  'axios': {
    singleton: false,         // Fiecare MFE poate avea versiunea lui
    requiredVersion: '^1.6.0',
  },

  // Company packages
  '@company/ui-kit': {
    singleton: true,          // Design system TREBUIE singleton
    requiredVersion: '^2.0.0',
  },

  // NU share:
  // - lodash (tree-shake per MFE e mai bun)
  // - moment/dayjs (trebuie configure per locale)
  // - Small utils (overhead > benefit)
}
```

### eager vs lazy

| Proprietate | Host | Remote | De ce |
|------------|------|--------|-------|
| **eager: true** | Da | Nu | Host-ul încarcă shared deps la start |
| **eager: false** | Nu | Da | Remote-ul le ia de la Host la runtime |

### Ce se întâmplă la Version Mismatch

```
Host are: vue@3.5.0 (singleton: true)
Remote cere: vue@^3.4.0

→ ^3.4.0 include 3.5.0 → Match! Remote folosește vue din Host

Host are: vue@3.5.0 (singleton: true)
Remote cere: vue@^3.6.0

→ ^3.6.0 NU include 3.5.0 →
  - strictVersion: false → Warning + Remote folosește vue din Host (may break)
  - strictVersion: true → ERROR, Remote nu se încarcă
```

### Best Practices

1. **Aliniază versiunile** - toate MFE-urile pe aceeași versiune Vue major.minor
2. **Folosește ranges** - `^3.4.0` nu `3.4.21` (mai flexibil)
3. **Update coordonat** - când upgrazi Vue, upgrazi toate MFE-urile
4. **Testare integration** - test cu versiunile exacte din producție
5. **Monitorizează** - log warnings de version mismatch

---

## 7. Routing în MFE

### Strategy 1: Host Deține Toate Rutele (RECOMANDAT)

Host-ul definește toate rutele. Remote-urile exportă doar componente.

```typescript
// HOST - router/index.ts
export const routes: RouteRecordRaw[] = [
  // Local routes
  { path: '/', component: HomeView },
  { path: '/about', component: AboutView },

  // MFE Products routes
  {
    path: '/products',
    component: createRemoteComponent(() => import('mfeProducts/ProductList'))
  },
  {
    path: '/products/:id',
    component: createRemoteComponent(() => import('mfeProducts/ProductDetail')),
    props: true
  },

  // MFE Checkout routes
  {
    path: '/checkout',
    component: createRemoteComponent(() => import('mfeCheckout/CheckoutPage'))
  },
]
```

**Pro:** Simplu, routing centralizat, fără conflicte
**Contra:** Host-ul trebuie updatat când se adaugă rute noi

### Strategy 2: Remote Exportă Rute

Remote-ul exportă configurația de rute. Host-ul le merge dinamic.

```typescript
// REMOTE - mfe-products/src/router/routes.ts (EXPOSED)
import type { RouteRecordRaw } from 'vue-router'

export const productRoutes: RouteRecordRaw[] = [
  {
    path: '/products',
    component: () => import('../components/ProductList.vue'),
  },
  {
    path: '/products/:id',
    component: () => import('../components/ProductDetail.vue'),
    props: true,
  },
]
```

```typescript
// HOST - dynamic route registration
import { useRouter } from 'vue-router'

async function registerRemoteRoutes() {
  const router = useRouter()

  try {
    const { productRoutes } = await import('mfeProducts/productRoutes')
    productRoutes.forEach(route => router.addRoute(route))
  } catch (error) {
    console.error('Failed to load product routes:', error)
  }

  try {
    const { checkoutRoutes } = await import('mfeCheckout/checkoutRoutes')
    checkoutRoutes.forEach(route => router.addRoute(route))
  } catch (error) {
    console.error('Failed to load checkout routes:', error)
  }
}
```

**Pro:** Remote-urile sunt mai autonome, adaugă rute fără a modifica host-ul
**Contra:** Mai complex, trebuie gestionat ordinea de loading

### Strategy 3: Catch-all Route cu Sub-router

```typescript
// HOST routes
{
  path: '/products/:pathMatch(.*)*',
  component: createRemoteComponent(() => import('mfeProducts/ProductApp')),
}
// Remote-ul are propriul router intern pentru sub-rute
```

**Pro:** Remote fully autonomous
**Contra:** URL sync complex, history management

### Navigation Guards în MFE

```typescript
// HOST - global guard (auth check)
router.beforeEach(async (to, from) => {
  const authStore = useAuthStore()

  if (to.meta.requiresAuth && !authStore.isAuthenticated) {
    return { path: '/login', query: { redirect: to.fullPath } }
  }

  if (to.meta.roles) {
    const hasRole = to.meta.roles.some(role => authStore.hasRole(role))
    if (!hasRole) return { path: '/403' }
  }
})
```

### Paralela cu Angular

| Aspect | Angular MFE Routing | Vue MFE Routing |
|--------|-------------------|-----------------|
| Lazy load remote | `loadRemoteModule()` → `loadChildren` | `defineAsyncComponent()` → `component` |
| Route config | Angular Route[] | RouteRecordRaw[] |
| Guards | `canActivate`, `canDeactivate` | `beforeEach`, `beforeEnter` |
| Dynamic routes | Not built-in (workaround) | `router.addRoute()` built-in |
| Complexity | Higher (NgModule/standalone) | Lower (just components) |

---

## 8. Comunicare Inter-MFE

### Pattern 1: Custom Events (RECOMANDAT)

Window CustomEvent - cel mai decuplat pattern. Funcționează cross-framework.

```typescript
// MFE Products - emite event când user adaugă în coș
window.dispatchEvent(new CustomEvent('cart:item-added', {
  detail: { productId: 123, name: 'Laptop', price: 2999 }
}))

// MFE Cart (sau Host) - ascultă event-ul
window.addEventListener('cart:item-added', ((event: CustomEvent) => {
  const { productId, name, price } = event.detail
  cartStore.addItem({ productId, name, price })
}) as EventListener)
```

Type-safe Event Bus implementation:
```typescript
// shared/eventBus.ts
type MFEEventMap = {
  'auth:login': { userId: string; name: string; role: string }
  'auth:logout': void
  'cart:item-added': { productId: number; name: string; price: number; quantity: number }
  'cart:updated': { totalItems: number; totalPrice: number }
  'notification:show': { message: string; type: 'success' | 'error' | 'warning' }
  'theme:changed': { theme: 'light' | 'dark' }
  'locale:changed': { locale: string }
}

class TypedEventBus {
  emit<K extends keyof MFEEventMap>(
    event: K,
    data: MFEEventMap[K]
  ): void {
    window.dispatchEvent(
      new CustomEvent(event, { detail: data })
    )
  }

  on<K extends keyof MFEEventMap>(
    event: K,
    handler: (data: MFEEventMap[K]) => void
  ): () => void {
    const listener = ((e: CustomEvent) => {
      handler(e.detail)
    }) as EventListener

    window.addEventListener(event, listener)

    // Return cleanup function
    return () => window.removeEventListener(event, listener)
  }
}

export const eventBus = new TypedEventBus()
```

Usage in Vue composable:
```typescript
// composables/useEventBus.ts
import { onUnmounted } from 'vue'
import { eventBus } from '@shared/eventBus'

export function useEventBus() {
  const cleanups: (() => void)[] = []

  function on<K extends keyof MFEEventMap>(
    event: K,
    handler: (data: MFEEventMap[K]) => void
  ) {
    const cleanup = eventBus.on(event, handler)
    cleanups.push(cleanup)
  }

  onUnmounted(() => {
    cleanups.forEach(fn => fn())
  })

  return { emit: eventBus.emit.bind(eventBus), on }
}

// In component
const { emit, on } = useEventBus()
on('cart:updated', ({ totalItems }) => {
  badgeCount.value = totalItems
})
```

### Pattern 2: localStorage / sessionStorage

```typescript
// MFE A - save state
const saveSharedState = (key: string, data: unknown) => {
  localStorage.setItem(`mfe:${key}`, JSON.stringify(data))
  // Dispatch storage event pentru alte tab-uri
  window.dispatchEvent(new StorageEvent('storage', {
    key: `mfe:${key}`,
    newValue: JSON.stringify(data),
  }))
}

// MFE B - listen for changes
window.addEventListener('storage', (event) => {
  if (event.key?.startsWith('mfe:')) {
    const data = JSON.parse(event.newValue || '{}')
    // Handle state change
  }
})
```

Pro: Persistent, cross-tab
Contra: Sync issues, no real-time, storage limits (5-10MB)

### Pattern 3: Shared Pinia Store

```typescript
// shared/stores/useSharedAuthStore.ts
// ATENȚIE: Creează coupling între MFE-uri!
export const useSharedAuthStore = defineStore('shared-auth', () => {
  const user = ref<User | null>(null)
  const token = ref<string | null>(null)
  const isAuthenticated = computed(() => !!user.value)

  return { user, token, isAuthenticated }
})

// Funcționează DOAR dacă pinia e singleton în Module Federation shared config
// shared: { pinia: { singleton: true } }
```

Când e OK: toate MFE-urile sunt Vue, aceeași versiune Pinia, auth state minimal
Când NU: MFE-uri multi-framework, state complex, frecvent updated

### Pattern 4: URL-based Communication

```typescript
// Navigation cu state
router.push({
  path: '/checkout',
  query: {
    productId: '123',
    quantity: '2'
  }
})

// Checkout MFE citește din URL
const route = useRoute()
const productId = computed(() => route.query.productId)
```

### Pattern 5: Broadcast Channel API (Modern)

```typescript
const channel = new BroadcastChannel('mfe-communication')

// Send
channel.postMessage({ type: 'cart:updated', payload: { items: 3 } })

// Receive
channel.onmessage = (event) => {
  if (event.data.type === 'cart:updated') {
    // Handle
  }
}
```

### Comparison Table

| Pattern | Coupling | Real-time | Cross-tab | Persistence | Best For |
|---------|----------|-----------|-----------|-------------|----------|
| **Custom Events** | Scăzut | Da | Nu | Nu | Majoritatea cazurilor |
| **localStorage** | Scăzut | Parțial | Da | Da | Preferințe, settings |
| **Shared Store** | Ridicat | Da | Nu | Opțional | Same-framework, auth state |
| **URL params** | Scăzut | Nu | N/A | Da (URL) | Navigation state |
| **BroadcastChannel** | Scăzut | Da | Da | Nu | Multi-tab sync |
| **PostMessage** | Scăzut | Da | N/A | Nu | iframe MFE-uri |

### Paralela cu Angular

Comunicarea inter-MFE e IDENTICĂ indiferent de framework. Custom Events, localStorage, URL params - toate sunt Web API-uri native. Diferența e doar în cum le wrappezi (composable în Vue vs service în Angular).

---

## 9. Error Handling și Resilience

### defineAsyncComponent cu Error Handling Complet

```typescript
import { defineAsyncComponent, h, ref } from 'vue'

// Helper function reutilizabil
function createResilientRemote(
  loader: () => Promise<any>,
  options: {
    name: string
    retries?: number
    timeout?: number
  }
) {
  const { name, retries = 3, timeout = 15000 } = options

  return defineAsyncComponent({
    loader: () => loadWithRetry(loader, retries),

    loadingComponent: {
      setup() {
        return () => h('div', { class: 'mfe-loading' }, [
          h('div', { class: 'spinner' }),
          h('p', `Se încarcă modulul ${name}...`)
        ])
      }
    },

    errorComponent: {
      props: ['error'],
      setup(props) {
        const retrying = ref(false)

        async function retry() {
          retrying.value = true
          window.location.reload()
        }

        return () => h('div', { class: 'mfe-error' }, [
          h('h3', `Modulul ${name} nu este disponibil`),
          h('p', props.error?.message || 'Eroare necunoscută'),
          h('button', {
            onClick: retry,
            disabled: retrying.value
          }, retrying.value ? 'Se reîncearcă...' : 'Reîncearcă')
        ])
      }
    },

    delay: 200,
    timeout,
  })
}

// Usage
const ProductList = createResilientRemote(
  () => import('mfeProducts/ProductList'),
  { name: 'Products', retries: 3, timeout: 10000 }
)
```

### Retry Logic cu Exponential Backoff

```typescript
async function loadWithRetry(
  loader: () => Promise<any>,
  maxRetries: number = 3,
  baseDelay: number = 1000
): Promise<any> {
  let lastError: Error

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await loader()
    } catch (error) {
      lastError = error instanceof Error ? error : new Error(String(error))

      if (attempt < maxRetries) {
        const delay = baseDelay * Math.pow(2, attempt) // 1s, 2s, 4s
        console.warn(
          `[MFE] Load attempt ${attempt + 1}/${maxRetries + 1} failed. ` +
          `Retrying in ${delay}ms...`,
          error
        )
        await new Promise(resolve => setTimeout(resolve, delay))
      }
    }
  }

  throw lastError!
}
```

### Circuit Breaker Pattern

```typescript
type CircuitState = 'closed' | 'open' | 'half-open'

class CircuitBreaker {
  private state: CircuitState = 'closed'
  private failures = 0
  private lastFailureTime = 0
  private successCount = 0

  constructor(
    private readonly name: string,
    private readonly failureThreshold: number = 3,
    private readonly resetTimeout: number = 30000,  // 30s
    private readonly halfOpenMaxAttempts: number = 1
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'open') {
      if (Date.now() - this.lastFailureTime >= this.resetTimeout) {
        this.state = 'half-open'
        this.successCount = 0
        console.log(`[CircuitBreaker:${this.name}] Half-open, testing...`)
      } else {
        throw new Error(
          `[CircuitBreaker:${this.name}] Circuit is OPEN. ` +
          `Retry after ${Math.ceil((this.resetTimeout - (Date.now() - this.lastFailureTime)) / 1000)}s`
        )
      }
    }

    try {
      const result = await fn()
      this.onSuccess()
      return result
    } catch (error) {
      this.onFailure()
      throw error
    }
  }

  private onSuccess() {
    if (this.state === 'half-open') {
      this.successCount++
      if (this.successCount >= this.halfOpenMaxAttempts) {
        this.state = 'closed'
        this.failures = 0
        console.log(`[CircuitBreaker:${this.name}] CLOSED (recovered)`)
      }
    } else {
      this.failures = 0
    }
  }

  private onFailure() {
    this.failures++
    this.lastFailureTime = Date.now()

    if (this.failures >= this.failureThreshold || this.state === 'half-open') {
      this.state = 'open'
      console.error(
        `[CircuitBreaker:${this.name}] Circuit OPENED after ${this.failures} failures`
      )
    }
  }

  getState(): CircuitState {
    return this.state
  }
}

// Usage per MFE
const breakers = {
  products: new CircuitBreaker('mfeProducts', 3, 30000),
  checkout: new CircuitBreaker('mfeCheckout', 3, 30000),
  dashboard: new CircuitBreaker('mfeDashboard', 3, 60000),
}

const ProductList = createResilientRemote(
  () => breakers.products.execute(() => import('mfeProducts/ProductList')),
  { name: 'Products' }
)
```

### Fallback Strategies

| Strategie | Când | Implementare |
|-----------|------|-------------|
| **Static message** | Default, orice MFE | ErrorComponent cu mesaj informativ |
| **Cached HTML** | Dacă user a vizitat înainte | Service Worker cache |
| **Feature degradation** | MFE opțional | Ascunde secțiunea, arată alternativă |
| **Redirect** | MFE are standalone URL | Link către MFE standalone |
| **Stale data** | Date non-critice | Arată datele din ultima sesiune |

### Monitoring & Alerting

```typescript
// composables/useMFEHealthCheck.ts
export function useMFEHealthCheck() {
  const mfeStatus = reactive<Record<string, 'healthy' | 'degraded' | 'down'>>({
    products: 'healthy',
    checkout: 'healthy',
    dashboard: 'healthy',
  })

  function reportFailure(mfeName: string, error: Error) {
    mfeStatus[mfeName] = 'degraded'

    // Send to monitoring
    sendToMonitoring({
      event: 'mfe_load_failure',
      mfe: mfeName,
      error: error.message,
      timestamp: Date.now(),
      userAgent: navigator.userAgent,
    })
  }

  function reportDown(mfeName: string) {
    mfeStatus[mfeName] = 'down'
    sendAlert(`MFE ${mfeName} is DOWN`, 'critical')
  }

  return { mfeStatus, reportFailure, reportDown }
}
```

---

## 10. CI/CD pentru MFE

### Independent Deployment Pipeline

```yaml
# azure-pipelines.yml - MFE Products
trigger:
  branches:
    include: [main, develop]
  paths:
    include:
      - mfe-products/**

pool:
  vmImage: 'ubuntu-latest'

variables:
  mfeName: 'mfe-products'
  nodeVersion: '20.x'

stages:
  - stage: Validate
    jobs:
      - job: LintAndTypeCheck
        steps:
          - task: NodeTool@0
            inputs: { versionSpec: '$(nodeVersion)' }
          - script: |
              cd $(mfeName)
              npm ci
              npm run lint
              npm run type-check
            displayName: 'Lint & Type Check'

  - stage: Test
    dependsOn: Validate
    jobs:
      - job: UnitTests
        steps:
          - script: |
              cd $(mfeName)
              npm ci
              npm run test -- --coverage
            displayName: 'Unit Tests'
          - task: PublishCodeCoverageResults@2
            inputs:
              summaryFileLocation: '$(mfeName)/coverage/cobertura-coverage.xml'

  - stage: Build
    dependsOn: Test
    jobs:
      - job: BuildMFE
        steps:
          - script: |
              cd $(mfeName)
              npm ci
              npm run build
            displayName: 'Build MFE'
          - task: PublishBuildArtifacts@1
            inputs:
              PathtoPublish: '$(mfeName)/dist'
              ArtifactName: '$(mfeName)'

  - stage: DeployStaging
    dependsOn: Build
    condition: eq(variables['Build.SourceBranch'], 'refs/heads/develop')
    jobs:
      - deployment: DeployStagingJob
        environment: 'staging'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureCLI@2
                  inputs:
                    azureSubscription: 'staging-sub'
                    scriptType: bash
                    scriptLocation: inlineScript
                    inlineScript: |
                      az storage blob upload-batch \
                        --source $(Pipeline.Workspace)/$(mfeName) \
                        --destination '$(mfeName)' \
                        --account-name stagingstorage \
                        --overwrite true

  - stage: IntegrationTest
    dependsOn: DeployStaging
    jobs:
      - job: SmokeTest
        steps:
          - script: |
              # Verifică că remoteEntry.js e accesibil
              HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
                https://staging.example.com/$(mfeName)/remoteEntry.js)
              if [ "$HTTP_STATUS" != "200" ]; then
                echo "remoteEntry.js not accessible! Status: $HTTP_STATUS"
                exit 1
              fi
              echo "MFE $(mfeName) is healthy on staging"
            displayName: 'Smoke Test'

  - stage: DeployProduction
    dependsOn: IntegrationTest
    condition: eq(variables['Build.SourceBranch'], 'refs/heads/main')
    jobs:
      - deployment: DeployProdJob
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureCLI@2
                  inputs:
                    azureSubscription: 'prod-sub'
                    scriptType: bash
                    scriptLocation: inlineScript
                    inlineScript: |
                      az storage blob upload-batch \
                        --source $(Pipeline.Workspace)/$(mfeName) \
                        --destination '$(mfeName)' \
                        --account-name prodstorage \
                        --overwrite true
                      # Invalidate CDN cache
                      az cdn endpoint purge \
                        --resource-group rg-mfe \
                        --profile-name mfe-cdn \
                        --name $(mfeName) \
                        --content-paths '/remoteEntry.js' '/*'
```

### Versioning Strategies

| Strategie | Cum | Pro | Contra |
|-----------|-----|-----|--------|
| **Content hash** | main.[hash].js | Cache efficient | remoteEntry.js trebuie updatat |
| **Semantic version** | v2.3.1/remoteEntry.js | Rollback ușor | Mai multe fișiere pe CDN |
| **Git hash** | [commit-hash]/remoteEntry.js | Traceable | Hash lung |
| **Timestamp** | [timestamp]/remoteEntry.js | Simplu | Nu e semantic |

**Recomandare:** Content hash pentru assets + remoteEntry.js la path fix (fără hash) cu Cache-Control: no-cache.

### Rollback Patterns

```
Rollback Flow:
1. Detectezi problemă (monitoring alert)
2. Identifici MFE-ul problematic
3. Rollback opțiuni:
   a) CDN: overwrite remoteEntry.js cu versiunea anterioară (30s)
   b) Pipeline: re-run deploy stage cu artifact-ul anterior
   c) Feature flag: disable MFE, arată fallback
4. Post-mortem + fix + redeploy
```

### Dynamic Remote URLs

```typescript
// config/remotes.ts
interface RemoteConfig {
  url: string
  scope: string
  module: string
}

const ENV_CONFIG: Record<string, Record<string, RemoteConfig>> = {
  production: {
    mfeProducts: {
      url: 'https://mfe-products.cdn.example.com/remoteEntry.js',
      scope: 'mfeProducts',
      module: './ProductList',
    },
    mfeCheckout: {
      url: 'https://mfe-checkout.cdn.example.com/remoteEntry.js',
      scope: 'mfeCheckout',
      module: './CheckoutPage',
    },
  },
  staging: {
    mfeProducts: {
      url: 'https://staging-products.example.com/remoteEntry.js',
      scope: 'mfeProducts',
      module: './ProductList',
    },
    // ...
  },
  development: {
    mfeProducts: {
      url: 'http://localhost:3001/remoteEntry.js',
      scope: 'mfeProducts',
      module: './ProductList',
    },
    // ...
  },
}

export function getRemoteConfig(name: string): RemoteConfig {
  const env = import.meta.env.MODE || 'development'
  return ENV_CONFIG[env]?.[name] || ENV_CONFIG.development[name]
}
```

---

## 11. Performanță în MFE

### Bundle Size per MFE

**Buget recomandat per MFE:**
- Initial JS: < 100KB gzipped (fără shared deps)
- Total cu shared deps (prima dată): < 300KB gzipped
- Subsequent MFE loads: < 100KB (shared deps deja cached)

### Lazy Loading Remotes

```typescript
// NU încarca toate MFE-urile la start
// Încarcă doar când user-ul navighează

// GREȘIT - eager loading
import ProductList from 'mfeProducts/ProductList'  // loaded immediately

// CORECT - lazy loading
const ProductList = defineAsyncComponent(() =>
  import('mfeProducts/ProductList')  // loaded on demand
)
```

### Preloading Strategies

```typescript
// Strategy 1: Prefetch on hover (most common)
function prefetchMFE(remoteUrl: string) {
  const link = document.createElement('link')
  link.rel = 'prefetch'
  link.href = remoteUrl
  document.head.appendChild(link)
}

// In navigation component
<RouterLink
  to="/products"
  @mouseenter="prefetchMFE('https://mfe-products.cdn.example.com/remoteEntry.js')"
>
  Products
</RouterLink>

// Strategy 2: Prefetch during idle time
if ('requestIdleCallback' in window) {
  requestIdleCallback(() => {
    prefetchMFE('https://mfe-products.cdn.example.com/remoteEntry.js')
  })
}

// Strategy 3: Prefetch based on user behavior
// Analytics: 80% of users go to /products after /
// → Prefetch products MFE when user is on home page
```

### Performance Monitoring

```typescript
// Measure MFE load time
const mfeLoadTimes: Record<string, number> = {}

function measureMFELoad(name: string, loader: () => Promise<any>) {
  return async () => {
    const start = performance.now()
    const module = await loader()
    const duration = performance.now() - start

    mfeLoadTimes[name] = duration

    // Report to analytics
    sendPerformanceMetric({
      metric: 'mfe_load_time',
      mfe: name,
      duration,
      timestamp: Date.now(),
    })

    if (duration > 3000) {
      console.warn(`[Performance] MFE ${name} took ${duration}ms to load`)
    }

    return module
  }
}

const ProductList = defineAsyncComponent({
  loader: measureMFELoad('products', () => import('mfeProducts/ProductList')),
  timeout: 15000,
})
```

### Caching Strategy

```
┌─────────────────┬──────────────────────┬────────────────────┐
│ Resource         │ Cache Strategy       │ De ce              │
├─────────────────┼──────────────────────┼────────────────────┤
│ remoteEntry.js  │ no-cache             │ Se schimbă la      │
│                  │ (always revalidate)  │ fiecare deploy     │
├─────────────────┼──────────────────────┼────────────────────┤
│ main.[hash].js  │ max-age=1y,          │ Hash se schimbă    │
│ chunk.[hash].js │ immutable            │ la content change  │
├─────────────────┼──────────────────────┼────────────────────┤
│ shared deps     │ max-age=1y,          │ Versioned, stable  │
│ (vue, pinia)    │ immutable            │                    │
├─────────────────┼──────────────────────┼────────────────────┤
│ index.html      │ no-cache             │ Entry point,       │
│                  │                      │ trebuie fresh      │
└─────────────────┴──────────────────────┴────────────────────┘
```

---

## 12. Native Federation (@softarc/native-federation)

### Ce e și De Ce Există

Module Federation e legat de Webpack. Dar ecosistemul Vue modern preferă **Vite** ca build tool. Problema: Vite nu suportă Module Federation nativ (Vite folosește Rollup, nu Webpack).

**Native Federation** e o alternativă creată de Manfred Steyer (@softarc) care:
- Funcționează cu **orice bundler** (Vite, esbuild, Webpack, Rollup)
- Bazată pe **ES Module imports** și **Import Maps** (standarde web)
- Nu depinde de Webpack runtime
- API similar cu Module Federation

### Cum Funcționează

```
Traditional Module Federation:
  Webpack Runtime → Container API → Shared Scope → Module Loading

Native Federation:
  Browser ESM → Import Maps → Dynamic import() → Module Loading
```

Import Maps (standard web):
```html
<script type="importmap">
{
  "imports": {
    "vue": "https://cdn.example.com/vue@3.5.0/dist/vue.esm-browser.js",
    "mfeProducts/": "https://mfe-products.example.com/"
  }
}
</script>
```

### Setup cu Vue + Vite

```typescript
// vite.config.ts - HOST
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import { withNativeFederation, shareAll } from '@softarc/native-federation/build'

export default defineConfig(async () => ({
  plugins: [
    vue(),
    await withNativeFederation({
      name: 'host',
      remotes: {
        'mfe-products': 'http://localhost:3001/remoteEntry.json',
      },
      shared: shareAll({
        singleton: true,
        strictVersion: true,
        requiredVersion: 'auto',
        includeSecondaries: false,
      }),
    }),
  ],
  build: {
    target: 'esnext',
  },
}))
```

```typescript
// vite.config.ts - REMOTE (mfe-products)
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import { withNativeFederation } from '@softarc/native-federation/build'

export default defineConfig(async () => ({
  plugins: [
    vue(),
    await withNativeFederation({
      name: 'mfe-products',
      exposes: {
        './ProductList': './src/components/ProductList.vue',
        './ProductDetail': './src/components/ProductDetail.vue',
      },
      shared: shareAll({
        singleton: true,
        strictVersion: true,
        requiredVersion: 'auto',
      }),
    }),
  ],
}))
```

### Loading remote în Host cu Native Federation

```typescript
// Host - main.ts
import { initFederation } from '@softarc/native-federation'

await initFederation({
  'mfe-products': 'http://localhost:3001/remoteEntry.json',
})

import('./bootstrap')
```

```typescript
// Host - loading remote component
import { loadRemoteModule } from '@softarc/native-federation'

const ProductList = defineAsyncComponent(() =>
  loadRemoteModule('mfe-products', './ProductList')
)
```

### Comparație: Webpack MF vs Native Federation

| Aspect | Webpack Module Federation | Native Federation |
|--------|--------------------------|-------------------|
| **Bundler** | Webpack 5 only | Any (Vite, esbuild, Webpack) |
| **Runtime** | Webpack runtime (~30KB) | Browser native ESM (~5KB) |
| **Manifest** | remoteEntry.js (JavaScript) | remoteEntry.json (JSON) |
| **Standard** | Webpack-specific | ES Modules + Import Maps |
| **Vite support** | Nu (fără hacks) | Da, nativ |
| **Maturity** | Matură (2020+) | Mai nouă (2023+) |
| **Community** | Mare | În creștere |
| **Config complexity** | Medie | Medie |
| **Browser support** | Toate (Webpack polyfills) | Modern browsers (ESM) |
| **Angular support** | @angular-architects/mf | @softarc/native-federation |
| **Vue support** | Direct (Webpack config) | @softarc/native-federation |
| **Performance** | Bună | Mai bună (no Webpack runtime) |

### Când Alegi Care

| Situație | Alegere | De ce |
|----------|---------|-------|
| Proiect nou cu Vite | **Native Federation** | Vite-native, no Webpack needed |
| Proiect existent cu Webpack | **Webpack MF** | Nu schimba bundler-ul |
| Echipa folosește Angular | **Native Federation** | @softarc/native-federation are Angular support excelent |
| Need max browser compat | **Webpack MF** | Webpack polyfills everything |
| Performance critical | **Native Federation** | Smaller runtime, ESM native |
| Proven at scale | **Webpack MF** | More battle-tested |

### Notă pentru interviu Arnia

JD-ul menționează Webpack 5 Module Federation. Dar e bine să menționezi Native Federation ca alternativă modernă - arată că ești la curent cu ecosistemul. "Folosesc Webpack MF pentru că e cerința proiectului, dar sunt conștient de Native Federation ca alternativă pentru proiecte noi cu Vite."

---

## 13. Paralela Completă: Angular MFE vs Vue MFE

### Configurare Side-by-Side

```typescript
// ANGULAR - Loading remote module
// Folosește @angular-architects/module-federation
const routes: Routes = [
  {
    path: 'products',
    loadChildren: () => loadRemoteModule({
      type: 'module',
      remoteEntry: 'http://localhost:3001/remoteEntry.js',
      exposedModule: './ProductModule',
    }).then(m => m.ProductModule),
  },
]

// VUE - Loading remote component
// Direct Webpack config, fără wrapper library
const routes: RouteRecordRaw[] = [
  {
    path: '/products',
    component: defineAsyncComponent(() =>
      import('mfeProducts/ProductList')
    ),
  },
]
```

### Comparison Table

| Aspect | Angular MFE | Vue MFE |
|--------|------------|---------|
| **Wrapper library** | @angular-architects/module-federation | Nu e nevoie (direct Webpack config) |
| **Config abstraction** | Angular CLI schematic | Manual webpack.config.js |
| **Remote loading** | loadRemoteModule() | defineAsyncComponent() + import() |
| **Lazy loading unit** | NgModule / Standalone component | Component (SFC) |
| **Routing** | loadChildren → Module routes | component → defineAsyncComponent |
| **Shared deps** | @angular/core, zone.js, rxjs | vue, vue-router, pinia |
| **Shared deps size** | ~200KB+ (Angular is larger) | ~50KB (Vue is lighter) |
| **DI in MFE** | Complex (multiple injectors) | Simple (provide/inject per MFE) |
| **Complexity** | Higher (Angular overhead) | Lower (Vue simpler) |
| **Build tool** | Angular CLI (Webpack) | Webpack 5 or Vite (cu Native Fed) |
| **CSS isolation** | ViewEncapsulation | Scoped styles / CSS Modules |
| **Error handling** | ErrorHandler service | app.config.errorHandler |
| **Bootstrap pattern** | Same (main.ts → bootstrap.ts) | Same (main.ts → bootstrap.ts) |

### Key Insights pentru Interviu

1. **Module Federation e identic** - e Webpack, nu Angular/Vue. Config-ul e 90% la fel.
2. **Vue MFE e mai simplu** - nu ai NgModule complexity, nu ai zone.js issues, componentele sunt self-contained.
3. **Bootstrap pattern e identic** - async boundary necesar în ambele.
4. **Shared deps diferă** - Angular shared deps sunt mai mari (Angular core ~200KB vs Vue ~50KB).
5. **Comunicare inter-MFE e identică** - Custom Events, localStorage, etc. sunt Web APIs, framework-agnostic.

---

## 14. Întrebări de Interviu MFE (20+)

### Q1: "Cum ai decide dacă proiectul are nevoie de MFE?"

Evaluez pe 3 axe: **team topology** (3+ echipe autonome → MFE), **deployment independence** (echipele au release cycles diferite → MFE), **domain boundaries** (domenii clar decuplate → MFE). Dacă niciuna nu se aplică, monolith sau monorepo e suficient. Red flags: echipă mică, MVP, domenii strâns cuplate. Am evaluat MFE pe un proiect Angular cu 4 echipe - am ales MFE pentru că coordonarea deployment-ului devenise bottleneck (2h merge conflicts, deploy sincronizat săptămânal).

### Q2: "Explică Module Federation - cum funcționează?"

Module Federation permite aplicațiilor Webpack să partajeze module la runtime. Host-ul consumă module de la Remote-uri. La navigare, Host fetch-uiește remoteEntry.js (manifestul remote-ului), negociază shared dependencies (vue, pinia ca singleton), apoi încarcă componenta expusă. Shared dependency negotiation e key: dacă Host are vue@3.5 și Remote cere vue@^3.4, se folosește versiunea Host-ului. Fără singleton: true, fiecare MFE ar avea propria instanță Vue → broken reactivity.

### Q3: "Cum configurezi shared dependencies?"

vue, vue-router, pinia TREBUIE să fie singleton: true. Fără asta, Vue nu funcționează corect cross-MFE. Folosesc requiredVersion cu semver ranges (^3.4.0), eager: true în Host și eager: false în Remote-uri. NU share tot - doar ce trebuie să fie singleton. axios, lodash - fiecare MFE poate avea versiunea lui. Monitorizez version mismatch warnings în development.

### Q4: "Cum gestionezi shared state între MFE-uri?"

Prefer Custom Events (window.dispatchEvent) pentru decuplare maximă. Implementez un TypedEventBus cu tipuri stricte. Pentru state persistent, folosesc localStorage cu StorageEvent pentru sync. Evit shared Pinia stores (tight coupling). Rule: minimizează shared state - dacă MFE-urile au nevoie de mult shared state, probabil boundaries sunt greșite.

### Q5: "Ce faci când un remote MFE e indisponibil?"

Trei niveluri: 1) defineAsyncComponent cu loadingComponent, errorComponent, timeout. 2) Retry logic cu exponential backoff (3 încercări: 1s, 2s, 4s). 3) Circuit breaker - după 3 failures în 5 min, stop trying, arată fallback direct. Fallback UI: mesaj informativ, cached version, sau feature degradation. Monitoring: alert la down > 2 min.

### Q6: "Cum asiguri consistența UI?"

Shared design system distribuit ca npm package (@company/ui-kit). Design tokens (CSS custom properties) pentru culori, spacing, typography. Visual regression testing (Chromatic) pe fiecare PR. Lint rules care enforces folosirea componentelor din design system. Trade-off: design system e o dependență shared - breaking changes afectează toate MFE-urile.

### Q7: "Cum ai migra de la monolith la MFE?"

Strangler Fig pattern: identifică cel mai independent module (DDD bounded context), extrage-l ca primul MFE, configurează Module Federation pe monolith (devine Host). Migrează gradual - nu big bang. Timeline realistă: 6-12 luni pentru app medie. Cel mai greu: refactorizarea shared state.

### Q8: "Cum faci deployment independent?"

Pipeline separat per MFE (Azure DevOps), path-based triggers. Build → test → deploy pe CDN separat. remoteEntry.js la URL fix, fără cache (Cache-Control: no-cache). Assets cu content hash (immutable cache). Rollback: overwrite remoteEntry.js cu versiunea anterioară (30 secunde).

### Q9: "Cum gestionezi routing-ul?"

Prefer "Host deține rutele" - simplu, centralizat. Host definește rute, lazy load remote components cu defineAsyncComponent. Alternativ: remote-urile exportă route configs, host-ul le merge cu router.addRoute(). Navigation guards în host pentru auth/authorization.

### Q10: "Ce metrici monitorizezi?"

Per MFE: load time, error rate, bundle size. Global: LCP, INP, CLS (Web Vitals). Infrastructure: CDN hit rate, remoteEntry.js availability. Business: deployment frequency per MFE, time to recovery. Alert pe: MFE load failure > 1%, load time > 3s, MFE down > 2min.

### Q11: "Cum gestionezi CSS isolation?"

Vue scoped styles (<style scoped>) sau CSS Modules. Design tokens via CSS custom properties (shared). BEM naming convention cu prefix per MFE (mfe-products__card). Shadow DOM dacă trebuie izolare totală. Nu share CSS global entre MFE-uri (doar prin design tokens).

### Q12: "Ce e bootstrap pattern și de ce e necesar?"

main.ts face import('./bootstrap') - async boundary. Fără asta, shared dependencies se încarcă eager și fiecare MFE primește propria copie de Vue (nu singleton). Bootstrap pattern permite Webpack-ului să negocieze shared deps ÎNAINTE de a rula codul aplicației. E identic în Angular MFE și Vue MFE.

### Q13: "Webpack Module Federation vs Native Federation?"

Webpack MF: proven, matur, funcționează cu Webpack 5. Native Federation: modern, funcționează cu Vite, bazat pe ESM/Import Maps. Dacă proiectul e pe Webpack (cum e la Arnia) → Webpack MF. Dacă proiect nou cu Vite → Native Federation. Ambele au API similar. Pot coexista (MFE-ul nu trebuie să știe ce folosește host-ul).

### Q14: "Cum testezi MFE-urile?"

Unit: composables, stores (Vitest). Component: Vue Test Utils. Integration: host + remote local (smoke tests). Contract: verifică interfața componentelor expuse. E2E: Cypress/Playwright pe staging (host + toate remotes live). CI: unit + component per MFE pipeline. Staging: integration + e2e.

### Q15: "Cum optimizezi performanța MFE?"

Lazy loading remotes (load pe route change). Prefetch pe hover/idle. Content hash caching (1y immutable). remoteEntry.js no-cache. Bundle budgets (< 100KB per MFE gzipped). Shared deps eliminate duplication. Preloading critical MFE-uri. Monitoring load times.

### Q16: "Cum gestionezi versioning?"

Semantic versioning per MFE. remoteEntry.js la URL fix (versiunea e internă). Assets cu content hash. Shared deps cu semver ranges în requiredVersion. Rollback = overwrite remoteEntry.js. Nu versioning explicit între MFE-uri (decuplat).

### Q17: "Care sunt cele mai mari challenges cu MFE?"

1) Shared dependency management (version alignment). 2) Debugging cross-MFE bugs. 3) UX consistency între echipe. 4) Complexitatea CI/CD. 5) Initial setup overhead. Mitigare: design system shared, centralized monitoring, pipeline templates, documentation.

### Q18: "Cum gestionezi auth în MFE?"

Auth în Host (single source of truth). Token în HttpOnly cookie sau memory. MFE-urile primesc auth state prin Custom Events sau shared store. Route guards în host verifică permissions ÎNAINTE de a încărca MFE-ul. Token refresh gestionat de host.

### Q19: "Cum asiguri type safety între MFE-uri?"

TypeScript declarations (declare module 'mfeProducts/ProductList'). Shared types package (@company/mfe-types). Contract testing: verifică că interfața expusă respectă tipurile. CI check: type-check pe remote ȘI pe host.

### Q20: "Experiența ta cu MFE - ce ai face diferit?"

Am lucrat cu Module Federation în Angular. Ce am învățat: 1) Start simplu (2-3 MFE-uri, nu 10). 2) Investește în design system ÎNAINTE de MFE migration. 3) Monitoring e crucial - fără el, debugging e imposibil. 4) Shared state trebuie minimizat. 5) Bootstrap pattern e non-negociabil. Aceste lecții se aplică identic în Vue.


---

**Următor :** [**06 - Arhitectura si Design Patterns** →](Vue/06-Arhitectura-si-Design-Patterns.md)