# Ghid de Pregătire - TypeScript / JavaScript / Node.js + Agentic Coding

> Materiale de pregătire pentru interviul din **9 martie 2026**.
> Poziția implică **agentic coding intensiv** (folosești AI ca partener de dezvoltare).
> Stack: **Node.js + Express** (backend) + **React** (frontend, bonus).
> Accentul se pune pe **TypeScript/JavaScript** și **modul de gândire**.

---

## Despre acest interviu

Intervievatorii vor să vadă:
1. **Cum gândești** — nu neapărat să știi totul pe de rost
2. **Cum folosești AI-ul** — ești confortabil cu agentic coding?
3. **TypeScript/JavaScript solid** — concepte avansate, nu sintaxă de bază
4. **Arhitectură Node.js/Express** — decizii de design, patterns, trade-offs
5. **React** — un plus, nu obligatoriu

**Insight cheie:** Pozițiile de agentic coding nu caută oameni care știu totul — caută oameni care știu **ce să ceară AI-ului, cum să verifice output-ul și cum să orchestreze soluții complexe**.

---

## Atuurile tale specifice pentru acest interviu

Înainte să studiezi, înțelege de ce ești un candidat puternic pentru acest job specific:

### Experiența Adobe Express este "golden ticket"-ul tău

Ai construit **UI pentru AI image generation** (Adobe Firefly) la una dintre cele mai mari companii de software din lume. Intervievatorii caută oameni care "înțeleg AI" — tu nu ai citit despre AI, ai **construit produse AI**.

Cum să o prezinți: nu ca "am lucrat la Adobe", ci ca "am construit interfețele prin care milioane de utilizatori interacționau cu AI generativ. Știu ce UX challenges apar, ce stări edge trebuie gestionate, cum echilibrezi latența cu experiența."

### Node.js nu e nou pentru tine

Ai folosit Node.js în producție la Adobe Express. Nu e un bullet point vag pe CV — e context real. Când intervievatorii întreabă de Node.js, nu zici "am citit despre asta", zici "l-am folosit la Adobe".

### Ești framework agnostic — și asta e rar

**Angular** (Arobs, Ivanti, Roboyo, ClinicDr) + **Vue** (Movalio) + **React Native** (Movalio/gosnippet) + **Lit** (Adobe) + **Stencil** (Haiilo) + **Ionic** (ClinicDr). Puțini developeri au traversat atâtea ecosisteme cu succes. Nu te "adaptezi" la un nou framework — **înțelegi conceptele independent de sintaxă**.

### Ai built real products cu real users

- **200+ clinici** în SUA (ClinicDr) — aplicație medicală critică
- **2000+ utilizatori** pe gosnippet.com (Movalio)
- **Milioane** de utilizatori Adobe Express
- **90% reducere birocrație** (CLIVE @ Arobs) — impact direct măsurabil

Nu ai lucrat doar în enterprise intern. Ai livrat produse cu utilizatori reali care au depins de ce ai construit tu.

### Ai mentorat și ai construit echipe

Program internship 6 luni la Odeen, 4 juniori mentorizați la Arobs, 8 juniori introduși în Angular la Accesa. Asta înseamnă că gândești sistematic — poți explica un concept clar și poți vedea "big picture" dincolo de linia de cod.

### Ce trebuie să ai grijă să nu subestimezi

- **"Nu știu React web"** — spune că ai React Native + înțelegi conceptele, nu că "nu știi React"
- **Node.js backend la scară** — ai folosit Node.js la Adobe, dar Express + arhitectură REST e ceva ce consolidezi acum. Fii onest, dar menționează că fundamentele le ai
- **"Nu am experiență cu agentic coding"** — dacă ai folosit Claude Code / Copilot chiar și pentru proiecte personale sau pregătire, e valid. Nu minimiza.

---

## Harta subiectelor de studiu

| Prioritate | Fișier | Topic | Timp estimat |
|-----------|--------|-------|-------------|
| **CRITICĂ** | [01-TypeScript-Avansat.md](./01-TypeScript-Avansat.md) | Type system, generics, utility types, narrowing | 3-4h |
| **CRITICĂ** | [02-JavaScript-Core-si-Event-Loop.md](./02-JavaScript-Core-si-Event-Loop.md) | Closures, prototypes, event loop, async/await, Promises | 3-4h |
| **CRITICĂ** | [04-Agentic-Coding-AI.md](./04-Agentic-Coding-AI.md) | AI workflows, prompt engineering, Claude/Copilot, MCP | 2h |
| **ÎNALTĂ** | [03-NodeJS-Express.md](./03-NodeJS-Express.md) | Node.js internals, Express middleware, API design, auth | 3h |
| **ÎNALTĂ** | [06-Gandire-si-Problem-Solving.md](./06-Gandire-si-Problem-Solving.md) | Cum gândești, algoritmică, live coding mindset | 2h |
| **ÎNALTĂ** | [05-Patterns-si-Arhitectura.md](./05-Patterns-si-Arhitectura.md) | Design patterns, arhitectură backend, clean code | 2h |
| **MEDIE** | [07-React-Overview.md](./07-React-Overview.md) | React hooks, state, componente (bonus) | 1-2h |
| **MEDIE** | [08-Intrebari-si-Raspunsuri.md](./08-Intrebari-si-Raspunsuri.md) | Q&A tipice, răspunsuri framework | 1h |

---

## Plan de studiu — 6 zile (3–9 martie 2026)

### Marți 3 martie — Fundamente JavaScript
| Bloc | Ce faci |
|------|---------|
| Dimineață | [02 - JavaScript Core](./02-JavaScript-Core-si-Event-Loop.md): closures, prototypes, `this` |
| După-amiază | [02] continuare: event loop, microtasks, async/await, Promises |
| Seară | Exersează: scrie cod fără AI, înțelege fiecare linie |

### Miercuri 4 martie — TypeScript Avansat
| Bloc | Ce faci |
|------|---------|
| Dimineață | [01 - TypeScript](./01-TypeScript-Avansat.md): generics, conditional types, mapped types |
| După-amiază | [01] continuare: utility types, decorators, declaration merging |
| Seară | Exersează: type challenges din [type-challenges](https://github.com/type-challenges/type-challenges) |

### Joi 5 martie — Node.js + Express
| Bloc | Ce faci |
|------|---------|
| Dimineață | [03 - Node.js](./03-NodeJS-Express.md): event loop Node, streams, module system |
| După-amiază | [03] continuare: Express middleware, routing, error handling, auth |
| Seară | Construiește un mini REST API cu Express + TypeScript |

### Vineri 6 martie — Agentic Coding + Arhitectură
| Bloc | Ce faci |
|------|---------|
| Dimineață | [04 - Agentic Coding](./04-Agentic-Coding-AI.md): workflows, cum prezinți folosirea AI |
| După-amiază | [05 - Patterns](./05-Patterns-si-Arhitectura.md): design patterns relevante pentru Node.js |
| Seară | [06 - Gândire](./06-Gandire-si-Problem-Solving.md): exersează live coding mindset |

### Sâmbătă 7 martie — React + Consolidare
| Bloc | Ce faci |
|------|---------|
| Dimineață | [07 - React](./07-React-Overview.md): hooks, state management, patterns |
| După-amiază | [08 - Q&A](./08-Intrebari-si-Raspunsuri.md): exersează răspunsurile cu voce tare |
| Seară | Review general: identifică zonele slabe |

### Duminică 8 martie — Review Final
| Bloc | Ce faci |
|------|---------|
| Dimineață | Recitește fișierele CRITICE (01, 02, 04) |
| După-amiază | Pregătește 3-5 povești STAR despre proiecte cu AI |
| Seară | Relaxare. Nu studia după ora 20:00. |

### Luni 9 martie — Ziua interviului
| Oră | Ce faci |
|-----|---------|
| Dimineață | Recitire rapidă a întrebărilor din [08-Q&A](./08-Intrebari-si-Raspunsuri.md) |
| Cu 1h înainte | Relaxare, o cafea |
| **Interviul** | **SUCCES!** |

---

## Ce caută intervievatorii (poziție cu agentic coding)

### Mentalitate "AI-first developer"

```
❌ Fără AI:  Scriu tot codul manual, memorez API-uri
✅ Cu AI:    Orchestrez soluții: știu CE să cer, VERIFIC output-ul, INTEGREZ rezultatele
```

### Semnale pozitive în interviu

- **"Am folosit Claude/Copilot/Cursor pentru X, dar am verificat că..."** — arată că folosești AI responsabil
- **"AI-ul a generat codul, dar eu am decis arhitectura pentru că..."** — tu rămâi în control
- **"Când AI-ul a greșit, am identificat problema prin..."** — debugging skills reale
- **"Am promptat iterativ: prima dată am cerut X, apoi am rafinat cu Y"** — prompt engineering
- **"Am scris testele înainte să generez implementarea cu AI"** — TDD + AI

### Ce NU vrea să audă

- "Copiez ce dă ChatGPT fără să înțeleg"
- "Nu știu de ce funcționează, AI-ul a scris-o"
- "Nu am folosit AI niciodată"

---

## TypeScript/JavaScript — Concepte cheie de nivel Senior

### JavaScript — Must know

| Concept | De ce contează |
|---------|---------------|
| **Closure** | Memory leaks, factory functions, module pattern |
| **Prototype chain** | `instanceof`, `Object.create`, inheritance |
| **Event loop** | Performance, deadlocks, microtasks vs macrotasks |
| **`this` binding** | Bugs comune, arrow vs regular functions |
| **Promise internals** | Chaining, error propagation, `Promise.all` vs `Promise.allSettled` |
| **Generators / Iterators** | Lazy evaluation, infinite sequences, async generators |
| **WeakMap / WeakRef** | Memory management, cache patterns |
| **Proxy / Reflect** | Metaprogramming, validation, observability |

### TypeScript — Must know

| Concept | De ce contează |
|---------|---------------|
| **Generics avansate** | Cod refolosibil, type safety fără any |
| **Conditional types** | `T extends U ? X : Y` — type-level logic |
| **Mapped types** | Transformarea tipurilor: `{ [K in keyof T]: ... }` |
| **Template literal types** | Type-safe string operations |
| **Discriminated unions** | Pattern matching, exhaustive checks |
| **Narrowing & type guards** | Cod sigur, fără cast-uri |
| **Infer keyword** | Extragerea tipurilor din alte tipuri |
| **Decorators** | Metaprogramming, NestJS-style |

---

## Node.js — Concepte cheie

| Concept | De ce contează |
|---------|---------------|
| **Event loop Node.js** | Diferit față de browser! `libuv`, phases |
| **Streams** | Procesare date mari fără OutOfMemory |
| **Worker Threads** | CPU-intensive tasks fără blocking |
| **Cluster module** | Utilizarea tuturor core-urilor CPU |
| **Module system** | CommonJS vs ESM, circular deps |
| **Error handling** | `try/catch` async, unhandledRejection, domains |
| **Memory management** | V8 heap, garbage collection, memory leaks |

---

## Agentic Coding — Ce înseamnă în practică

### Flux tipic agentic coding

```
1. Tu definești PROBLEMA și CONTEXTUL (README, spec, constraints)
2. AI generează soluția inițială
3. Tu REVIZUIEȘTI: logică, securitate, edge cases
4. Tu RAFINEZI: feedback la AI pentru iterații
5. Tu VALIDEZI: rulezi teste, verifici în browser/Postman
6. Tu INTEGREZI: PR review, merge, deployment
```

### Unelte de agentic coding pe care le cunoști

- **Claude Code** (ce folosești acum) — agentic, cu acces la filesystem și terminal
- **GitHub Copilot** — autocomplete + chat în IDE
- **Cursor** — IDE construit pentru AI pair programming
- **Continue.dev** — open source alternative la Copilot
- **Aider** — CLI tool pentru agentic coding
- **MCP (Model Context Protocol)** — protocol pentru a conecta AI la tools externe

---

## Sfaturi specifice pentru acest interviu

1. **Gândește cu voce tare** — intervievatorii vor să vadă procesul, nu doar răspunsul
2. **Pune întrebări de clarificare** — "Avem constrângeri de performanță?" — semn de maturitate
3. **Recunoaște ce nu știi, dar arată cum ai afla** — "Nu am lucrat cu X, dar aș verifica Y"
4. **Demonstrează că folosești AI inteligent** — nu "generez și copiez", ci "orchestrez și verific"
5. **Trade-offs mereu** — la orice soluție, menționează ce ai sacrificat
6. **Experiența ta Angular/Vue e un avantaj** — TypeScript, patterns, arhitectura — toate se aplică

---

## Legături rapide

- [01 - TypeScript Avansat](./01-TypeScript-Avansat.md)
- [02 - JavaScript Core & Event Loop](./02-JavaScript-Core-si-Event-Loop.md)
- [03 - Node.js + Express](./03-NodeJS-Express.md)
- [04 - Agentic Coding & AI](./04-Agentic-Coding-AI.md)
- [05 - Patterns & Arhitectură](./05-Patterns-si-Arhitectura.md)
- [06 - Gândire & Problem Solving](./06-Gandire-si-Problem-Solving.md)
- [07 - React Overview (bonus)](./07-React-Overview.md)
- [08 - Întrebări & Răspunsuri](./08-Intrebari-si-Raspunsuri.md)

---

*Creat: 3 martie 2026 — interviu: 9 martie 2026 — 6 zile de pregătire*

**Următor:** [**01 - TypeScript Avansat →**](./01-TypeScript-Avansat.md)
