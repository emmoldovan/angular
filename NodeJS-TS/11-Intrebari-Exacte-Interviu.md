# 11 - Întrebările Exacte ale Interviului

> Lista completă de întrebări primite de la intervievator, cu răspunsuri complete
> ancorate în experiența ta reală. Studiază asta cu 1-2 zile înainte de interviu.
> Exersează cu voce tare — nu doar citești.

---

## FRONT-END

---

**Q: How do you structure a Next.js application for rapid MVP delivery while keeping it maintainable?**

> "Pentru MVP rapid și mentenabil, folosesc App Router cu o structură feature-based.
> Nu pornesc cu 5 layers de abstractizare — pornesc simplu și adaug complexitate pe măsură
> ce e nevoie.
>
> Concret: Server Components by default (nu adaugi 'use client' decât când trebuie),
> shadcn/ui pentru componente gata, Zustand pentru state simplu, React Query pentru
> server state, Route Handlers ca BFF simplu.
>
> Structura de foldere: `src/features/[feature]/` conține tot ce ține de acel feature —
> nu `src/components/`, `src/services/`, `src/hooks/` separat (acelea devin greu de navigat
> rapid). Expoziez un `index.ts` per feature care definește public API-ul.
>
> Cel mai important pentru MVP: avoid over-engineering. Nu construiești infrastructure pentru
> scale pe care nu-l ai. Refactorizezi când ai dovezi că e necesar."

---

**Q: When would you choose SSR vs SSG vs CSR in Next.js?**

> "În App Router, Server Components sunt default și acoperă SSR — rulează pe server,
> au acces direct la DB, nu trimit JavaScript la client.
>
> **Server Components (SSR):** orice pagină cu date personalizate per user: dashboard,
> profil, orice ce depinde de autentificare. Date fresh la fiecare request.
>
> **SSG cu generateStaticParams:** pagini cu date statice sau semi-statice: blog, marketing,
> documentație. Generezi la build time. Cu `revalidate: 3600` ai ISR — revalidezi periodic.
>
> **Client Components (CSR):** doar când ai nevoie de interactivitate: click handlers,
> useState, browser APIs, real-time updates. Regula: 'use client' cât mai jos în arbore.
>
> La Adobe Express, majority of features were SSR/SSG — am înțeles direct impactul
> pe performance când trimiți mai puțin JS la client pe mobile."

---

**Q: How do you handle shared state in a growing MVP (Context, Zustand, Redux, server state)?**

> "Depinde de natura state-ului. Eu separ în două categorii: UI state și server state.
>
> **Server state** (date de pe API/DB): React Query. Gestionează loading, error, caching,
> invalidare după mutații. Nu duplicezi în store local ce ai deja pe server.
>
> **UI state local:** useState. Dacă nu trebuie partajat, nu-l muți sus.
>
> **UI state partajat simplu:** Context API — bun pentru theme, locale, auth user.
> Atenție: orice schimbare cauzează re-render pe toți consumatorii.
>
> **UI state complex:** Zustand — fără Provider, fără boilerplate, selectori granulari
> (re-render doar când valoarea selectată se schimbă). La un MVP în creștere e prima
> mea alegere față de Redux — ai același rezultat cu 10x mai puțin cod.
>
> Redux l-aș alege doar dacă echipa are deja experiență cu el sau dacă am nevoie de
> time-travel debugging și Redux DevTools."

---

**Q: How do you debug and optimize a slow React component?**

> "Procesul meu în 3 pași:
>
> **1. Identify:** React DevTools Profiler — înregistrez interacțiunea lentă, văd flame chart-ul.
> Identific ce componente re-renderează inutil (fără props schimbate).
>
> **2. Diagnose — cauze comune:**
> - Funcție recreată la fiecare render pasată ca prop → `useCallback`
> - Obiect/array recreat la fiecare render → `useMemo` sau constant în afara componentei
> - Child re-renderează chiar dacă props nu s-au schimbat → `React.memo`
> - Computație costisitoare (filtrare/sortare pe array mare) → `useMemo`
>
> **3. Fix specific:** Nu adaug memo/useCallback everywhere — are cost. Măsor înainte,
> aplic după.
>
> Pentru liste de 1000+ iteme: react-window pentru virtualizare. Pentru bundle size:
> dynamic imports + React.lazy. La Adobe am lucrat direct pe mobile performance —
> am redus bundle size semnificativ prin modularizare și code splitting."

---

**Q: How do you test components that rely on async data fetching?**

> "Stack-ul meu: Vitest + React Testing Library + MSW (Mock Service Worker).
>
> MSW interceptează fetch/axios la nivel de network — nu mockezi module, interceptezi
> requesturi reale. Asta înseamnă că testezi comportamentul real al componentei, nu
> implementarea internă.
>
> Definești handlers: `http.get('/api/users', () => HttpResponse.json([...]))`.
> Testezi 3 stări: loading, success, error. Pentru error, overrideezi handler-ul cu
> `server.use(http.get('/api/users', () => HttpResponse.error()))` doar pentru acel test.
>
> Principiu cheie: `retry: false` în QueryClient la teste (altfel retriezi de 3 ori
> și testul e lent). `resetHandlers()` după fiecare test pentru izolare.
>
> `await waitFor(() => expect(screen.getByText('Emanuel')).toBeInTheDocument())` —
> aștepți până apare, nu te bazezi pe timing."

---

## BACK-END

---

**Q: How do you handle input validation and error normalization?**

> "Validare la boundary-uri — nu în business logic intern. Folosesc Zod pentru schema
> validation și un middleware dedicat.
>
> Middleware validate(schema) apelează `schema.safeParse(req.body)`. Dacă fail →
> returnez 400 cu `error.flatten()` care structurează erorile per field: util pentru
> frontend să afișeze erori inline pe formular.
>
> Pentru normalizare erori: o ierarhie de clase `AppError extends Error` cu statusCode
> și code (machine-readable). Error handler global cu 4 parametri diferențiază erori
> operaționale (NotFoundError, ValidationError) de bug-uri programatice. Erori
> operaționale → statusCode specific. Bug-uri → 500, log cu stack trace, mesaj generic
> în producție (nu expui detalii interne).
>
> correlationId pe fiecare request — apare în response și în logs. Dacă utilizatorul
> raportează o eroare, ai imediat contextul din logs."

---

**Q: Explain the Node.js event loop and common performance pitfalls.**

> "Node.js e single-threaded dar non-blocking. Event loop-ul coordonează între call stack
> și queue-uri. Ordinea: cod sincron → microtasks (Promise.then, process.nextTick) →
> macrotasks (setTimeout, I/O callbacks, setImmediate).
>
> Fazele event loop Node (libuv): timers → pending callbacks → idle/prepare → poll
> (așteaptă I/O — faza principală) → check (setImmediate) → close callbacks.
>
> **Performance pitfalls:**
>
> 1. **CPU-intensive pe main thread** — un `fibonacci(45)` sincron blochează TOATE
> requesturile pe durata lui. Fix: Worker Threads sau job queue.
>
> 2. **Await în for loop** — execuție serială când vrei paralelism.
> `for await (const id of ids) fetch(id)` = N requesturi serial.
> Fix: `Promise.all(ids.map(id => fetch(id)))`.
>
> 3. **Memory leaks** — event listeners adăugate fără cleanup, closures care captează
> referințe mari, global variables accumulate.
>
> 4. **Sync file I/O** — `fs.readFileSync` în handler blochează event loop-ul.
> Folosești `fs.promises.readFile` sau streams."

---

**Q: How do you prevent blocking the main thread?**

> "Regula principală: tot ce durează mai mult de câteva milisecunde trebuie să fie
> async sau delegat.
>
> **I/O operations:** async by default în Node — `await db.query()`, `await fs.readFile()`.
> Niciodată variante sync în request handlers.
>
> **CPU-intensive tasks** (image processing, criptografie grea, ML inference, parsing
> JSON de sute de MB): Worker Threads. Creezi un worker, trimiți datele, primești
> rezultatul prin message passing. Main thread rămâne liber.
>
> **Operații lungi periodice:** setImmediate sau process.nextTick pentru a ceda controlul
> event loop-ului între iterații dacă procesezi batches mari.
>
> **AI calls:** niciodată sincron cu timeout lung. Async Job + Polling sau SSE —
> descris în detaliu la întrebarea despre HTTP timeout."

---

**Q: How do you handle prompt versioning and evaluation?**

> "Tratez prompturile ca pe cod: versionat în Git, review process, eval suite înainte
> de deployment.
>
> Fiecare prompt e un obiect cu version, name, template function. Am un registry care
> mapează la versiunea activă. Schimbările de prompt = PR cu justificare.
>
> Evaluarea: un set de cazuri de test cu input + expected output + evaluator function.
> Rulez eval suite în CI când se schimbă un prompt. Dacă pass rate scade sub threshold
> (ex: 90%), deployment e blocat.
>
> Pentru logging în producție: Langfuse sau similar — loghez prompt, output, latență,
> cost per request. Pot vedea regresia în timp dacă un prompt se degradează.
>
> AB testing pe prompturi: hash user ID → 20% primesc varianta nouă. Compar metrics
> (user satisfaction, task completion rate) înainte de roll-out complet."

---

**Q: How do you avoid hallucination issues in production systems?**

> "Nu există soluție 100%, dar poți reduce semnificativ riscul cu mai multe straturi:
>
> **1. Constrain output format:** JSON mode / Zod validation pe output. Dacă AI-ul
> returnează ceva în afara schema → eroare controlată, nu garbage afișat utilizatorului.
>
> **2. RAG în loc de knowledge from training:** Nu întrebi AI-ul să 'știe' — îi dai
> contextul explicit. 'Răspunde DOAR pe baza documentelor de mai jos. Dacă răspunsul
> nu e în documente, spune că nu știi.'
>
> **3. Prompt engineering strict:** 'Do NOT invent information. If uncertain, say so.'
> Explicit instructions bate training behavior.
>
> **4. Output validation cu un al doilea model:** Generat răspunsul cu modelul principal,
> validat cu un al doilea prompt: 'Este răspunsul următor factual consistent cu contextul dat?'
>
> **5. Human-in-the-loop** pentru cazuri high-stakes: output-ul AI e o sugestie, userul
> confirmă înainte de acțiune.
>
> La Adobe am văzut direct ce înseamnă când AI generează ceva neașteptat — UI-ul trebuie
> să gestioneze elegant orice output, nu să presupună că e perfect."

---

## INFRA / DEVOPS

---

**Q: How would you enforce coverage thresholds?**

> "Configurat în vitest.config.ts cu `coverage.thresholds`: linii, branch-uri, funcții.
> Dacă orice metric e sub threshold → `exit code 1` → CI/CD pipeline se oprește.
>
> ```typescript
> // vitest.config.ts
> coverage: {
>   thresholds: { lines: 80, branches: 75, functions: 85 },
>   exclude: ['**/*.test.ts', 'src/types/**', 'src/mocks/**']
> }
> ```
>
> În GitHub Actions: `vitest run --coverage` ca step separat, înainte de deployment.
> Pull request nu poate fi merged dacă coverage scade.
>
> Filosofie: threshold-urile nu sunt scopul — sunt minimul. Nu scrii teste ca să ajungi
> la 80%, scrii teste pentru că ai confidence în cod. Coverage e un proxy, nu o garanție."

---

**Q: How would you scale AI request handling if traffic increases suddenly?**

> "Mai multe straturi de scalare, în ordine:
>
> **1. Caching semantic:** Prompturi identice sau similare → returnezi rezultatul cacheuit.
> Reduce costul și latența pentru cazuri comune.
>
> **2. Queue cu rate limiting:** BullMQ cu rate limiter configurat la limita provider-ului.
> Request-urile se queue-uiesc în loc să fie reject-ate. Back-pressure controlat.
>
> **3. Multiple AI providers cu fallback:** OpenAI rate limit atins → redirect la Anthropic
> sau Groq. Circuit breaker pe fiecare provider.
>
> **4. Model routing:** Task simplu → model ieftin și rapid (GPT-4o-mini). Task complex →
> model puternic. Reduci cost și latență pentru majority of requests.
>
> **5. Horizontal scaling al worker-ilor:** Queue e shared, poți adăuga oricâți workeri.
> Kubernetes HPA bazat pe queue depth."

---

## AI

---

**Q: Describe your daily AI coding workflow.**

> *(Răspunsul tău personal — varianta recomandată)*
>
> "Dimineața, înainte să încep, mă asigur că AI-ul are contextul actualizat — CLAUDE.md
> sau echivalentul, cu convențiile proiectului.
>
> Pentru fiecare task: definesc problema și constraints eu — nu las AI-ul să ghicească
> contextul. Cer structura inițială sau codul cu context specific din proiect.
> Revizuiesc critic: logică de business, securitate, edge cases, compatibilitate cu
> arhitectura existentă. Iterez cu feedback specific.
>
> Am și perspectiva celuilalt sens: la Adobe am construit UI-ul pentru AI image generation —
> am livrat un produs AI utilizatorilor, nu doar am folosit AI ca tool. Asta îmi dă o
> înțelegere mai completă despre ce înseamnă AI în producție."

---

**Q: How do you maintain AI instruction Markdown docs?**

> "Tratez CLAUDE.md / .cursorrules ca documentație living — versionat în Git, actualizat
> când adaug o convenție nouă sau când observ că AI-ul generează ceva greșit față de
> standardele proiectului.
>
> Conținut: tech stack, naming conventions, pattern-uri preferate, ce să evite, exemple
> de cod conform standardelor. Suficient de specific ca AI-ul să genereze cod care
> se integrează direct, nu care trebuie refactorizat.
>
> Feedback loop: dacă AI-ul generează ceva incompatibil cu arhitectura noastră, asta e
> un semnal că documentul trebuie actualizat. Revizuim la sprint review.
>
> Nu îl fac prea lung — AI-ul are context window limitat și un document de 50 pagini
> dilueaza instrucțiunile importante."

---

**Q: How would you build a tool-based coding agent?**

> "Un agent = LLM + loop + tools. Construiești un agentic loop:
>
> 1. Trimiți task-ul utilizatorului la LLM cu lista de tools disponibile (read_file,
> write_file, run_bash, search_code etc.) — fiecare cu description și schema parametrilor.
>
> 2. LLM decide ce tool să apeleze și cu ce parametri.
>
> 3. Tu execuți tool-ul și returnezi rezultatul la LLM.
>
> 4. Repeți până LLM returnează `finish_reason: 'stop'` — a terminat task-ul.
>
> Aspecte cheie: max iterations pentru a preveni loop-uri infinite, error handling pe
> fiecare tool call, logging pentru debugging, human-in-the-loop pentru operații
> distructive (delete, push to prod).
>
> Claude Code e exact asta — un agent cu tools: read/write files, bash, browser via MCP."

---

**Q: What is MCP and when would you use it?**

> "Model Context Protocol — un standard open creat de Anthropic pentru a conecta AI
> models la external tools și data sources. Ca USB pentru AI: un port standard în loc
> de integrări custom per IDE.
>
> Un MCP Server expune tools, resources și prompts. AI Client-ul (Claude, Cursor, agent
> custom) apelează tools-urile prin protocol standardizat — JSON-RPC.
>
> Când îl folosesc: când vreau să dau AI-ului acces la sisteme specifice proiectului —
> DB schema, API intern, documentație proprietară. Construiesc un MCP server care
> expune aceste resurse și orice AI client compatible le poate folosi imediat.
>
> Avantaj față de hardcoding tools: dacă schimb IDE-ul sau AI client-ul, MCP server-ul
> rămâne același."

---

**Q: How do you evaluate an AI agent's output automatically?**

> "Depinde de tipul output-ului:
>
> **Output structurat (JSON, cod):** Validare cu schema (Zod) — dacă nu respectă schema,
> fail automat. Pentru cod generat: rulez linter + tests. Dacă tests fail → output reject.
>
> **Output text/natural language:** Mai greu. Abordări:
> - LLM-as-judge: al doilea prompt evaluează primul output: 'Este acest răspuns corect,
>   complet și în conformitate cu instrucțiunile? Răspunde cu score 1-5 și justificare.'
> - Eval suite cu cazuri cunoscute (ground truth) — ca unit tests pentru prompts.
> - Human baseline: o dată, evaluezi manual 100 de cazuri. Compari outputs noi cu baseline.
>
> **Metrici pe care le urmăresc în producție:**
> - Task completion rate (a reușit să completeze ce i s-a cerut?)
> - Consistency (același input → output similar?)
> - Latency și cost per request
> - Error rate (outputs reject-ate de schema validator)"

---

## ARCHITECTURE

---

**Q: If product direction changes weekly, how do you keep architecture adaptable?**

> "Câteva principii pe care le aplic consistent:
>
> **Feature flags:** Lansez incremental, pot roll back fără deploy. Nu merge o
> features pentru toți utilizatorii odată — controlezi gradual. La Adobe, release
> cadence era rapidă și feature flags erau esențiale.
>
> **Modular architecture:** Fiecare feature e independentă, comunică prin interfețe.
> Schimbi implementarea unui modul fără să atingi celelalte.
>
> **Low coupling:** Dacă checkout importă direct din payments, e coupling strâns.
> Checkout depinde de `PaymentProcessor` interface — poți schimba provider-ul fără
> să atingi checkout logic.
>
> **Avoid over-optimization:** Nu construiesc pentru scale pe care nu-l am. Asta e
> cea mai mare capcană în MVP — petreci timp pe infrastructure pentru 1M users când
> ai 100. Refactorizezi cu dovezi, nu cu presupuneri.
>
> **Refactor cadence:** 20% din sprint pentru technical debt. Nu lași să se acumuleze.
> Architecture Decision Records pentru decizii importante — știi de ce ai ales ceva,
> e mai ușor să schimbi când contextul se schimbă."

---

## DEEP TECHNICAL

---

**Q: If an HTTP request times out when communicating with an AI agent, what solution would you implement?**

> **Răspunsul complet — include toate cele 3 opțiuni:**
>
> "Prima regulă: nu blochez request lifecycle-ul. AI calls pot dura 60+ secunde —
> dacă aștepți sincron, timeout-ul lovi utilizatorii chiar dacă AI-ul lucrează corect.
>
> **Opțiunea A — Async Job + Polling** (default-ul meu):
> Clientul trimite POST, primește imediat `{ jobId: 'abc123' }` cu status 202.
> Un background worker procesează AI task-ul. Clientul poll-uiește GET /jobs/abc123
> la fiecare 2-3 secunde. Când e done → primește rezultatul.
> Avantaje: funcționează cu orice client HTTP, simplu de implementat, robust.
>
> **Opțiunea B — Server-Sent Events / Streaming:**
> Clientul deschide o conexiune SSE la GET /api/ai/stream.
> Backend-ul streamează tokens pe măsură ce LLM-ul le generează.
> Clientul vede răspunsul token by token — ca ChatGPT.
> Nu mai există timeout — conexiunea e keep-alive.
> Avantaje: UX excelent, latentă percepută mică.
>
> **Opțiunea C — Message Queue:**
> Pentru sisteme distribuite la scară: push task în BullMQ/SQS/RabbitMQ.
> Workers independenți procesează. Retry automat pe eroare.
> Back-pressure când AI provider e lent.
>
> Alegerea depinde de context: pentru MVP aleg A (simplu, funcționează),
> pentru UX bun adaug B (streaming), pentru scale aleg C."

---

## Ordinea de studiat

Dacă ai timp limitat, prioritizează în această ordine:

| Prioritate | Întrebare | Fișier |
|-----------|-----------|--------|
| **1** | HTTP timeout cu AI agent | [10 - AI în Producție](./10-AI-in-Productie.md) |
| **2** | SSR vs SSG vs CSR | [09 - Next.js](./09-NextJS-Frontend-Avansat.md) |
| **3** | Tool-based coding agent | [10 - AI în Producție](./10-AI-in-Productie.md) |
| **4** | Prompt versioning & hallucination | [10 - AI în Producție](./10-AI-in-Productie.md) |
| **5** | State management (Zustand + React Query) | [09 - Next.js](./09-NextJS-Frontend-Avansat.md) |
| **6** | MCP | [10 - AI în Producție](./10-AI-in-Productie.md) |
| **7** | Next.js structure pentru MVP | [09 - Next.js](./09-NextJS-Frontend-Avansat.md) |
| **8** | Event loop + performance pitfalls | [02 - JavaScript Core](./02-JavaScript-Core-si-Event-Loop.md) |
| **9** | Debug React component lent | [09 - Next.js](./09-NextJS-Frontend-Avansat.md) |
| **10** | Architecture adaptabilă | [10 - AI în Producție](./10-AI-in-Productie.md) |

---

*[← 10 - AI în Producție](./10-AI-in-Productie.md) | [← Ghid principal](./00-Ghid-Pregatire.md)*
