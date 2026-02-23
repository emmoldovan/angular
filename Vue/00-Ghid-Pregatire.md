# Ghid de Pregătire - Senior Frontend Architect (Vue.js - Arnia Software)

> Materiale complete de pregătire pentru interviul la **Arnia Software** pe poziția de
> **Senior Frontend Architect - Federated Platform**. Stack: Vue 3 (Composition API),
> Module Federation (Webpack 5), Pinia, TypeScript, Azure DevOps, BFF C#/.NET.
> Documentul include paralele Vue vs Angular la fiecare concept major.

---

## Despre acest ghid

Acest ghid este creat pentru Emanuel Moldovan, care are experiență solidă pe Angular (Principal Engineer level) și se pregătește pentru un interviu Vue.js. Fiecare secțiune include:

- **Teorie** - conceptele fundamentale
- **Cod** - snippets TypeScript + Vue 3 SFC cu explicații detaliate
- **Paralela Angular vs Vue** - la fiecare concept major
- **Trade-offs** - decizii arhitecturale cu argumente pro/contra
- **Întrebări de interviu** - cu răspunsuri la nivel architect

**Context important:** La interviul de Angular nu te-ai descurcat bine pe zona de micro-frontenduri. Fișierul `05-Micro-Frontenduri-Module-Federation.md` este cel mai detaliat și trebuie studiat cel mai atent.

---

## Tech Stack Arnia (din Job Description)

| Tehnologie | Detalii |
|-----------|---------|
| **Frontend** | Vue 3 (Composition API), TypeScript |
| **State Management** | Pinia |
| **Arhitectura** | Micro-frontenduri cu Module Federation (Webpack 5) |
| **Build Tool** | Webpack 5 (pentru MFE), Vite (pentru dev) |
| **CI/CD** | Azure DevOps Pipelines |
| **IaC** | Azure Bicep |
| **Backend** | BFF pattern cu C#/.NET |
| **Testare** | Vitest, Vue Test Utils |
| **Alte cerinte** | Scalabilitate, performanță, mentoring echipă |

---

## Harta subiectelor de studiu

| Prioritate | Fișier | Topic | Timp Estimat |
|-----------|--------|-------|-------------|
| **CRITICAL** | [05-Micro-Frontenduri-Module-Federation.md](./05-Micro-Frontenduri-Module-Federation.md) | Module Federation, MFE architecture, comunicare inter-MFE | 3-4h |
| **CRITICAL** | [01-Vue3-Core-Composition-API.md](./01-Vue3-Core-Composition-API.md) | Composition API, ref/reactive, lifecycle, template syntax | 2-3h |
| **HIGH** | [02-Reactivitate-Vue-vs-Angular.md](./02-Reactivitate-Vue-vs-Angular.md) | Sistem reactiv Vue 3 (Proxy-based), ref, computed, watch | 1-2h |
| **HIGH** | [04-State-Management-Pinia.md](./04-State-Management-Pinia.md) | Pinia stores, composition stores, storeToRefs | 1-2h |
| **HIGH** | [12-Intrebari-Interviu-Architect.md](./12-Intrebari-Interviu-Architect.md) | Întrebări architect + răspunsuri framework | 2h |
| **HIGH** | [06-Arhitectura-si-Design-Patterns.md](./06-Arhitectura-si-Design-Patterns.md) | Composables, smart/dumb, folder structure | 1-2h |
| **MEDIUM** | [03-Dependency-Injection-Provide-Inject.md](./03-Dependency-Injection-Provide-Inject.md) | provide/inject, Symbol keys, app-level provides | 1h |
| **MEDIUM** | [08-TypeScript-in-Vue.md](./08-TypeScript-in-Vue.md) | Generic components, typed props/emits | 1h |
| **MEDIUM** | [10-DevOps-CICD-Azure.md](./10-DevOps-CICD-Azure.md) | Azure DevOps pipelines, Bicep, deployment MFE | 1h |
| **MEDIUM** | [11-BFF-CSharp-dotNET.md](./11-BFF-CSharp-dotNET.md) | BFF pattern, C#/.NET basics | 30min |
| **LOW** | [07-Performanta-si-Optimizare.md](./07-Performanta-si-Optimizare.md) | Virtual DOM, lazy loading, Vapor Mode | 30min |
| **LOW** | [09-Testare-Vue.md](./09-Testare-Vue.md) | Vitest, Vue Test Utils | 30min |
| **LOW** | [13-Pitch-Personal-Vue.md](./13-Pitch-Personal-Vue.md) | Cum prezinți experiența Angular ca relevantă pentru Vue | 30min |

---

## Plan de studiu pe 3 zile

### Ziua 1 (Duminică) - Fundamentale Vue + MFE Start
| Bloc | Durată | Ce faci |
|------|--------|---------|
| Dimineață | 2-3h | [01-Vue3-Core-Composition-API.md](./01-Vue3-Core-Composition-API.md) - Composition API, ref/reactive, lifecycle |
| După-amiază | 2h | [02-Reactivitate-Vue-vs-Angular.md](./02-Reactivitate-Vue-vs-Angular.md) - Sistemul reactiv |
| Seară | 2h | [05-Micro-Frontenduri-Module-Federation.md](./05-Micro-Frontenduri-Module-Federation.md) - Prima jumătate (teorie + config) |

### Ziua 2 (Luni) - MFE Deep Dive + State + Architecture
| Bloc | Durată | Ce faci |
|------|--------|---------|
| Dimineață | 2h | [05-Micro-Frontenduri-Module-Federation.md](./05-Micro-Frontenduri-Module-Federation.md) - A doua jumătate (comunicare, error handling, CI/CD) |
| După-amiază | 1.5h | [04-State-Management-Pinia.md](./04-State-Management-Pinia.md) |
| După-amiază | 1.5h | [06-Arhitectura-si-Design-Patterns.md](./06-Arhitectura-si-Design-Patterns.md) |
| Seară | 1h | [03-Dependency-Injection-Provide-Inject.md](./03-Dependency-Injection-Provide-Inject.md) |

### Ziua 3 (Marți) - Consolidare + Interviu Prep
| Bloc | Durată | Ce faci |
|------|--------|---------|
| Dimineață | 2h | [12-Intrebari-Interviu-Architect.md](./12-Intrebari-Interviu-Architect.md) - Exersează răspunsurile |
| După-amiază | 1h | [08-TypeScript-in-Vue.md](./08-TypeScript-in-Vue.md) |
| După-amiază | 1h | [10-DevOps-CICD-Azure.md](./10-DevOps-CICD-Azure.md) + [11-BFF-CSharp-dotNET.md](./11-BFF-CSharp-dotNET.md) |
| Seară | 1h | [13-Pitch-Personal-Vue.md](./13-Pitch-Personal-Vue.md) + Review [05](./05-Micro-Frontenduri-Module-Federation.md) |

---

## Vue 3.5+ Features Overview (Ce e nou)

### Vue 3.5 (Septembrie 2024)
- **Reactive Props Destructure** - `const { modelValue } = defineProps()` - acum reactive by default
- **useTemplateRef()** - API mai clar pentru template refs
- **Deferred Teleport** - `<Teleport defer>` pentru lazy teleport
- **useId()** - generare unica de ID-uri (util pentru SSR/accessibility)
- **onWatcherCleanup()** - cleanup logic în watchers

### Vue 3.4 (Decembrie 2023)
- **defineModel()** - simplificarea v-model pe componente custom
- **Improved type inference** pentru `defineProps` și `defineEmits`
- **v-bind same-name shorthand** - `:id` în loc de `:id="id"`

### Vue 3.3 (Mai 2023)
- **Generic Components** - `<script setup lang="ts" generic="T">`
- **defineSlots()** - typed slots
- **defineOptions()** - `inheritAttrs`, `name` fără script separat

---

## De ce experiența Angular e un AVANTAJ

Ca Angular developer experimentat, ai deja:

| Concept Angular | Echivalent Vue | Ce știi deja |
|----------------|---------------|-------------|
| Components + Templates | SFC (Single File Components) | Componente, data binding, lifecycle |
| Signals (`signal()`) | `ref()` / `reactive()` | Reactivitate fină, dependențe automate |
| `computed()` | `computed()` | Valori derivate cu cache |
| `effect()` | `watchEffect()` | Side effects reactive |
| Services + DI | Composables + `provide/inject` | Logică partajată, ierarhie DI |
| NgRx / BehaviorSubject | Pinia | State management centralizat |
| Standalone Components | Toate componentele Vue sunt "standalone" | Import explicit al dependențelor |
| `@if`, `@for` | `v-if`, `v-for` | Control flow în templates |
| `@Input()` / `@Output()` | `defineProps()` / `defineEmits()` | Comunicare părinte-copil |
| `ng-content` | `<slot>` | Content projection |
| Lazy loading routes | Lazy loading routes (identic conceptual) | Code splitting |
| Module Federation (Angular) | Module Federation (Vue) | **Aceeași tehnologie Webpack 5!** |

**Key insight:** Module Federation este o tehnologie **Webpack**, nu Angular sau Vue. Configurarea este aproape identică. Diferența e doar în cum încarci componentele remote (Angular: `loadRemoteModule` → Vue: `defineAsyncComponent` + dynamic import).

---

## Sfaturi pentru interviu

1. **Nu te scuza că nu știi Vue** - în schimb, arată că înțelegi conceptele și poți face tranziția rapid
2. **Folosește paralele** - "În Angular fac X cu Y, și înțeleg că în Vue echivalentul este Z"
3. **Focus pe arhitectură** - pentru o poziție de Architect, deciziile de design contează mai mult decât sintaxa
4. **MFE knowledge is framework-agnostic** - Module Federation, deployment strategies, comunicare inter-MFE sunt aceleași indiferent de framework
5. **Demonstrează curiozitate** - pune întrebări despre setup-ul lor specific, challenges pe care le au

---

## Legături rapide

- [01 - Vue 3 Core & Composition API](./01-Vue3-Core-Composition-API.md)
- [02 - Reactivitate Vue vs Angular](./02-Reactivitate-Vue-vs-Angular.md)
- [03 - Dependency Injection (provide/inject)](./03-Dependency-Injection-Provide-Inject.md)
- [04 - State Management cu Pinia](./04-State-Management-Pinia.md)
- [05 - Micro-Frontenduri & Module Federation](./05-Micro-Frontenduri-Module-Federation.md) ⭐ PRIORITAR
- [06 - Arhitectură și Design Patterns](./06-Arhitectura-si-Design-Patterns.md)
- [07 - Performanță și Optimizare](./07-Performanta-si-Optimizare.md)
- [08 - TypeScript în Vue](./08-TypeScript-in-Vue.md)
- [09 - Testare Vue](./09-Testare-Vue.md)
- [10 - DevOps, CI/CD, Azure](./10-DevOps-CICD-Azure.md)
- [11 - BFF cu C#/.NET](./11-BFF-CSharp-dotNET.md)
- [12 - Întrebări Interviu Architect](./12-Intrebari-Interviu-Architect.md)
- [13 - Pitch Personal Vue](./13-Pitch-Personal-Vue.md)


---

**Următor :** [**01 - Vue 3 Core & Composition API** →](Vue/01-Vue3-Core-Composition-API.md)