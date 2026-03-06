# 13 - Scoring Rubric, Red Flags & Toate Întrebările

> Fișierul cel mai important pentru a NU fi eliminat.
> Știi CE să spui, dar mai ales CE SĂ NU SPUI.
> Scoring: 9 dimensiuni, 1-5 fiecare. Hire: 32+. Strong hire: 38+.

---

## Scoring Rubric — ce ești notat

| # | Dimensiune | Ce evaluează | Notă |
|---|-----------|-------------|------|
| 1 | **Frontend Depth** | Trade-off thinking, experiență reală vs răspunsuri memorate | /5 |
| 2 | **Backend / Node.js** | Poate proiecta pentru workload-uri AI-heavy? Async patterns? | /5 |
| 3 | **AI Workflow ⚠️ CRITIC** | Are un workflow REAL zilnic? Poate face demo? Tools specifice + obiceiuri? | /5 |
| 4 | **AI Instruction Docs** | Creează/menține markdown instructions pentru consistență AI? | /5 |
| 5 | **Architecture Thinking** | Poate proiecta pentru schimbare? Feature flags, adapters, clean boundaries? | /5 |
| 6 | **Problem Decomposition** | Poate descompune cerințe vagi în task-uri executabile? | /5 |
| 7 | **Communication** | Clar, direct, explică trade-off-uri bine? | /5 |
| 8 | **Testing Discipline** | Testele fac parte din workflow, nu un afterthought? | /5 |
| 9 | **Coding Assignment** | (Vezi 12-Coding-Assignments.md) | /5 |
| | **TOTAL** | **Hire: 32+. Strong hire: 38+** | **/45** |

**Insight critic din Research Sheet:** Dimensiunea 3 (AI Workflow) este marcată CRITICĂ — dacă pici aici, restul contează puțin pentru acest rol.

---

## Interview Flow — 75 minute timeline

```
0-3 min:   WARM-UP
           Întrebare: "Walk me through your current AI coding setup and daily workflow."
           ⚠️ Aceasta e prima întrebare. Nu e small talk — e evaluare.
           Ei setează tonul: "Nu testăm cunoștințe de carte, testăm cum lucrezi."

3-8 min:   AI WORKFLOW DEEP DIVE (CELE MAI IMPORTANTE 5 MINUTE)
           Întrebările 11-15. Aleg 2-3 bazate pe răspunsul tău la warm-up.
           Dacă menționezi Cursor → întreabă despre .cursorrules
           Dacă menționezi Claude Code → întreabă despre CLAUDE.md
           "Dacă pică aici, restul abia mai contează pentru acest rol."

8-18 min:  FRONTEND + BACKEND
           2 întrebări frontend + 2 backend bazate pe CV-ul tău.
           Adaptează dificultatea sus/jos după cum răspunzi.
           Folosesc follow-up probes dacă vor să adâncească.

18-23 min: ARCHITECTURE
           Întrebările 16-17. Focus pe ambiguitate și cerințe care se schimbă.
           "Candidații buni se aprind aici — au war stories."

23-25 min: TRANZIȚIE LA CODING
           Explică assignment-ul. Pui întrebări de clarificare.
           ⚠️ "Cum pui întrebări despre brief spune multe. Clarifici scope? Întrebi despre constraints?"

25-65 min: LIVE CODING (30-40 min)
           Screen share ON. Tu construiești cu AI tools proprii.
           Ei observă în tăcere. Intervin doar dacă ești blocat >5 min.
           ⚠️ CHEIE: Observă FOLOSIREA AI, nu doar codul.

65-70 min: CODE WALKTHROUGH
           "Walk me through ce ai construit. De ce aceste decizii?
           Ce ai îmbunătăți cu mai mult timp? Ce ai schimba pentru producție?"

70-75 min: WRAP-UP
           Întrebările tale despre rol/echipă/companie.
           ⚠️ "Întrebările candidaților dezvăluie prioritățile lor."
```

---

## FRONTEND — Toate întrebările cu Strong Signals & Red Flags

---

### Q1 — Migrare Next.js de la CSR la SSR/SSG

**Întrebarea exactă:** "You inherit a Next.js app where every page is client-rendered with useEffect + fetch. The PM wants better SEO and faster first paint. Walk me through your migration strategy — what stays CSR, what moves to SSR/SSG, and how do you decide?"

**✅ Strong Signals:**
- Gândești în trade-offs, nu absolut
- Date fresh → SSR. Static marketing → SSG. User-specific dashboard → CSR cu skeleton. Produse → ISR cu revalidation
- Știi de React Server Components și când ajută

**❌ Red Flags:**
- "Mut totul pe SSR" (fără trade-off thinking)
- Nu menționezi cache invalidation sau revalidation
- Nu consideri data layer deloc

**Follow-up probe:** *"What breaks when you SSR a component that uses `window` or `localStorage`?"*

**Răspunsul tău:**
> "window și localStorage nu există pe server — sunt browser APIs. Un component care le accesează direct în body va crăpa la SSR cu ReferenceError. Fix-uri: (1) `typeof window !== 'undefined'` check, (2) muti în useEffect (rulează doar pe client), (3) `dynamic(() => import('./Component'), { ssr: false })` — Next.js nu renderizează componenta pe server deloc. Prefer varianta 3 pentru componente care sunt fundamental client-only."

---

### Q2 — 47 re-renders la un singur caracter

**Întrebarea exactă:** "A React component re-renders 47 times when you type one character in a search input. You have React DevTools open. Walk me through exactly how you diagnose and fix this — step by step."

**✅ Strong Signals:**
- Deschide Profiler, identifică ce componente re-renderează și de ce
- Verifică: context providers care wrappează prea mult, memoization lipsă, inline object/function props, state ridicat prea sus
- Menționează React.memo, useMemo, useCallback cu înțelegere clară CÂND ajută (nu "folosești mereu")

**❌ Red Flags:**
- "Adaug React.memo peste tot" fără să explici de ce
- Nu poți explica ce declanșează re-renders
- Nu menționezi Profiler
- "Folosesc Redux" ca fix

**Follow-up probe:** *"When does React.memo actually make performance WORSE?"*

**Răspunsul tău:**
> "React.memo face shallow comparison la fiecare render al părintelui. Dacă props-urile includ funcții sau obiecte create inline (fără useCallback/useMemo), comparația eșuează mereu → re-render oricum, plus costul comparației. De asemenea, pentru componente triviale (un span cu un text), costul memo > costul re-render-ului. Regula: memo-izezi componente care sunt scumpe de randat SAU care primesc props stabile."

---

### Q3 — Dashboard real-time AI job status (50 jobs concurrent)

**Întrebarea exactă:** "You need to build a dashboard that shows real-time AI job status — pending, processing, completed, failed — for ~50 concurrent jobs. What's your state management approach and why?"

**✅ Strong Signals:**
- WebSocket/SSE pentru real-time updates
- State normalizat (jobs by ID, nu array)
- React Query / SWR cu polling fallback
- Separare server state vs client state (UI state în Zustand/useState, job data în React Query)
- Optimistic UI pentru acțiunile userului

**❌ Red Flags:**
- Totul în Redux fără raționale
- Nu menționează transport real-time
- Polling la 500ms ca singura opțiune
- Amestecă server state și UI state

**Follow-up probe:** *"How do you handle the case where a WebSocket disconnects and 5 jobs changed status while offline?"*

**Răspunsul tău:**
> "La reconnect: fetch starea curentă a tuturor job-urilor printr-un REST call (reconciliere), nu mă bazez că am primit toate evenimentele. WebSocket e 'best effort' — dacă pierzi conectivitatea, NU ai garanție că ai primit toate mesajele. Reconciliere la reconnect e pattern-ul corect. Practic: `useEffect` pe `ws.onopen` care face un `queryClient.invalidateQueries(['jobs'])` pentru a forța refetch complet."

---

### Q4 — Form wizard cu AI endpoint de 8-15s la step 3

**Întrebarea exactă:** "You're building a form wizard (5 steps) where step 3 calls an AI endpoint that takes 8-15 seconds. How do you handle the UX so users don't abandon?"

**✅ Strong Signals:**
- Streaming partial results dacă posibil
- Progress indicators care "trăiesc" (nu un spinner static)
- Allow users să meargă înapoi în timp ce așteaptă
- Optimistic navigation
- Cancel/retry cu AbortController
- Pre-fetch la step 2 dacă input-urile sunt deja cunoscute

**❌ Red Flags:**
- Arată doar un loading spinner 15 secunde
- Nu consideră perceived performance
- Nu menționează cancellation sau error recovery

**Follow-up probe:** *"What happens if the user navigates away and comes back — is the result still there?"*

**Răspunsul tău:**
> "Dacă navigarea e în-app (router push), job-ul continuă pe backend — stochezi jobId în state/URL param. La revenire, verifici status: dacă e done, afișezi rezultatul din cache/backend. Dacă e still processing, reattach-ezi la SSE stream sau polling. Dacă navigarea e full page refresh, depinde de persistență — pentru MVP: jobId în localStorage sau URL search param."

---

### Q5 — Testare component cu IntersectionObserver + WebSocket + AI streaming

**Întrebarea exactă:** "How do you test a component that depends on an IntersectionObserver, a WebSocket connection, and calls an AI streaming endpoint?"

**✅ Strong Signals:**
- Mockezi fiecare dependință la granița ei
- IntersectionObserver mock în test setup
- WebSocket via mock server sau mock class
- AI streaming → mock ReadableStream
- Testezi comportamentul, nu implementarea
- MSW pentru API mocking

**❌ Red Flags:**
- "Nu testez componente complexe" sau "Doar E2E"
- Nu poate articula ce mockezi vs ce testezi
- Nu menționează edge cases (disconnect, timeout, partial stream)

**Follow-up probe:** *"Do you prefer `userEvent` or `fireEvent`, and why?"*

**Răspunsul tău:**
> "`userEvent` — simulează interacțiune reală a utilizatorului (focus, keydown, keyup, input events în ordine), nu doar dispatch de event. `fireEvent` e sync și minimal — util pentru cazuri simple sau când `userEvent` e overkill. În general: `userEvent.setup()` pentru forms și interactions complexe, `fireEvent` pentru simple trigger events."

---

## BACKEND — Toate întrebările cu Strong Signals & Red Flags

---

### Q6 — /health timeout din cauza AI proxy

**Întrebarea exactă:** "Your Node.js API proxies requests to an LLM that takes 5-30 seconds to respond. Under load, the API starts timing out even for fast endpoints like /health. What's happening and how do you fix it?"

**✅ Strong Signals:**
- Înțelege event loop blocking vs saturation
- Long-running HTTP connections consumă connection pool
- Soluții: separă AI proxy în serviciu/worker separat, streaming pentru a răspunde devreme, request queuing cu backpressure, circuit breakers
- Știe de libuv thread pool limits

**❌ Red Flags:**
- "Măresc timeout-ul"
- "Adaug mai multă RAM"
- Nu poate explica de ce /health e afectat de AI calls lente
- Nu înțelege connection pool exhaustion

**Follow-up probe:** *"How would you monitor event loop lag in production?"*

**Răspunsul tău:**
> "`perf_hooks.monitorEventLoopDelay()` din Node.js pentru histogram al lag-ului. Sau `clinic.js` (clinic doctor, clinic flame) pentru profiling complet. În producție: metrici custom — înregistrezi timestamp înainte și după `setImmediate`, diferența e event loop lag. Dacă lag > 100ms consistent, ai o problemă. Expui ca metric Prometheus/Datadog."

---

### Q7 — API contract pentru document → AI analysis (20-60s)

**Întrebarea exactă:** "Design the API contract for: client submits a document, backend sends it to AI for analysis (takes 20-60s), client needs progress updates and the final result. Show me the endpoints and the flow."

**✅ Strong Signals:**
- `POST /jobs` → returnează jobId imediat (202 Accepted)
- `GET /jobs/:id` pentru polling SAU WebSocket/SSE pentru push
- Job states: queued → processing → completed/failed
- Stochează rezultatul persistent
- Idempotency key la submission, retry semantics, TTL pe rezultate
- Webhook callback option pentru B2B

**❌ Red Flags:**
- Single synchronous POST care blochează 60 secunde
- Fără job abstraction
- Fără error states
- Returnează 200 pentru orice

**Follow-up probe:** *"What if the worker crashes mid-processing — how does the job recover?"*

**Răspunsul tău:**
> "Job state machine: dacă workerul crăpă în mijlocul procesării, job-ul rămâne în starea 'processing'. La startup sau periodic (cron), un 'stale job detector' găsește job-uri stuck în processing mai mult de N minute și le resetează la 'queued' pentru retry. BullMQ face asta automat cu `stalled` jobs — dacă workerul nu trimite heartbeat, job-ul e re-queued."

---

### Q8 — Validare input pentru AI prompt (prompt injection)

**Întrebarea exactă:** "You're validating user input that will be injected into an AI prompt. What's your validation strategy beyond basic type checking?"

**✅ Strong Signals:**
- Input length limits (prompt injection via mega-input)
- Sanitizare special tokens/delimiters care pot manipula promptul
- Rate limiting per user
- Content validation (no executable code injection)
- Output validation — verifici AI response înainte să trimiți la client
- Zod/joi schemas + semantic validation

**❌ Red Flags:**
- Doar Joi pentru type validation și gata
- Nu știe de prompt injection
- Nu validează output-ul

**Follow-up probe:** *"How would you detect if someone is trying to jailbreak your AI endpoint through crafted inputs?"*

**Răspunsul tău:**
> "Câteva straturi: (1) Pattern matching pe input pentru fraze comune de jailbreak ('ignore previous instructions', 'you are now', 'DAN mode'). (2) Un 'guard model' — un prompt secundar rapid care evaluează dacă input-ul încearcă să manipuleze sistemul. (3) Output scanning — dacă AI-ul returnează ceva din afara domeniului așteptat (ex: cod de programare când aștepți analiză de sentiment), reject și log. (4) Rate limiting agresiv pe utilizator suspect."

---

### Q9 — Monorepo CI când se schimbă un shared validation schema

**Întrebarea exactă:** "Your team uses a monorepo with 3 Next.js apps and 5 shared packages. A junior dev's PR changes a shared validation schema. How should the CI pipeline handle this?"

**✅ Strong Signals:**
- Affected package detection (Turborepo/Nx)
- Rulezi tests doar pentru pachete schimbate + dependentele lor
- Schema changes trigger: type checking pe toți consumatorii, integration tests, snapshot tests pentru API contracts
- Pre-merge: lint, type, unit, integration. Post-merge: deploy doar apps afectate

**❌ Red Flags:**
- Rulează toate testele de fiecare dată
- Nu înțelege affected graph
- Nu menționează cum propagă schema changes

**Follow-up probe:** *"How do you prevent a shared package change from shipping a breaking change to production without anyone noticing?"*

**Răspunsul tău:**
> "Semantic versioning strict pe shared packages cu changesets (Changesets tool). Breaking changes → major version bump → CI fail dacă nu e bumped. Type checking strict în CI pentru toți consumatorii. Optional: API contract testing cu Pact sau similar. Changesets forțează ca autorul să declare explicit că e o breaking change înainte de merge."

---

### Q10 — 50MB PDF upload → AI extraction (sub 2 minute)

**Întrebarea exactă:** "You need to handle file uploads (up to 50MB PDFs) that get sent to an AI for extraction. The user expects results in under 2 minutes. Design the flow from upload to result delivery."

**✅ Strong Signals:**
- Multipart upload → stream la S3/blob storage (NU bufferezi în memorie)
- Crezi un job record. Worker descarcă fișierul, îl chunking-uiește dacă e nevoie
- Trimite la AI. Stream progress înapoi via SSE
- Stochezi rezultatul extracted
- Validare fișier (type, size, malware scan), chunking pentru AI context limits, parallel processing pe chunks, cost tracking per user

**❌ Red Flags:**
- Încarci 50MB în memorie
- Fără streaming
- Fără storage — procesezi inline în request handler
- Fără chunking strategy

**Follow-up probe:** *"What if the PDF is 200 pages — does your AI call change?"*

**Răspunsul tău:**
> "Da, semnificativ. 200 pagini depășesc context window-ul oricărui model actual (GPT-4o: ~128k tokens ≈ 100 pagini dense). Strategy: chunk în secțiuni de 20-30 pagini, procesezi în paralel, combini rezultatele. Pentru extracție structurată, poți face map-reduce: fiecare chunk extrage entitățile din secțiunea lui, un final call 'reduce' combină și deduplicates. Cost și latență cresc liniar — importantă transparența față de user."

---

## AI WORKFLOW — Toate întrebările (CELE MAI CRITICE)

---

### Q11 — Demo AI setup live

**Întrebarea exactă:** "Show me your AI coding setup right now. What tools do you have open? Walk me through what happens from the moment you get a Jira ticket to your first PR."

**✅ Strong Signals:**
- Are un workflow real, practicat
- Citește ticket → descompune în subtask-uri → deschide Cursor/Claude Code/Copilot → scrie/rafinează prompt → generează cod → revizuiește critic → rulează tests → iterează
- Menționează: custom rules files, .cursorrules, CLAUDE.md sau similar
- Are opinii despre care tool pentru care task

**❌ Red Flags:**
- Vag: "Folosesc ChatGPT uneori"
- Fără workflow structurat
- Nu poate numi tools specifice sau explica când le folosește pe unele vs altele
- Nu menționează validarea după ce AI generează cod

**Follow-up probe:** *"Show me a prompt you'd write to scaffold a new API endpoint. What context do you include?"*

**Răspunsul tău:**
> "Context: tech stack și versiunile (Next.js 14 App Router, TypeScript, Prisma, Zod), convențiile din proiect (asyncHandler wrapper, AppError class, validate middleware), un exemplu de endpoint existent pentru stil. Task: specific — ce endpoint, ce face, ce validări. Constraints: ce NU trebuie să facă (ex: nu hash-ui parola — asta e în service). Example output format dacă e neclar. Nu cer 'generează un endpoint' — cer 'generează un PATCH /users/:id endpoint urmând pattern-ul din [fișier]'."

---

### Q12 — Review checklist pentru cod generat de AI

**Întrebarea exactă:** "You asked AI to generate a React component and it looks correct — renders fine, passes the basic test. But something feels off. What's your review checklist before you commit?"

**✅ Strong Signals:**
- Verifică: accessibility (aria, keyboard nav), error boundaries, edge cases (empty state, loading, error), memory leaks (cleanup în useEffect), unnecessary re-renders, TypeScript types proprii (nu `any` peste tot), error handling real (nu doar console.log), test coverage pe unhappy paths
- Citește codul linie cu linie — nu se bazează pe output

**❌ Red Flags:**
- "Dacă trece testul, e bine"
- Nu face review manual
- Are încredere oarbă în AI output
- Nu poate articula ce AI-ul greșește des

**Follow-up probe:** *"What's a pattern AI consistently generates poorly that you always have to fix?"*

**Răspunsul tău:**
> "Câteva pattern-uri constante pe care le-am observat: (1) useEffect cleanup — AI uită `return () => subscription.unsubscribe()` sau `controller.abort()`. (2) Error boundaries — generează happy path perfect, ignoră error states. (3) Loading states granulare — pune un singur `isLoading`, nu distinge între 'loading first time' și 're-fetching'. (4) TypeScript `any` în locuri de edge — AI pune `as any` când nu știe tipul exact în loc să rezolve corect. (5) Accessibility — generează `<div onClick>` în loc de `<button>`, uită `aria-label` pe icons."

---

### Q13 — Setarea AI instruction docs pentru echipă

**Întrebarea exactă:** "You're starting a new Next.js project for the team. How do you set up AI instruction docs (Markdown) so that every developer on the team gets consistent AI-generated code?"

**✅ Strong Signals:**
- Creează: project conventions doc (naming, file structure, import patterns), architecture decisions doc (state management, API patterns), prompt templates pentru task-uri comune (component nou, endpoint nou, test nou), anti-patterns doc (ce să NU facă)
- Le pune în repo root, le referențiază în .cursorrules sau CLAUDE.md
- Iterează bazat pe ce AI greșește
- Tratează ca living docs — PR reviews le actualizează

**❌ Red Flags:**
- Nu a auzit de AI instruction docs
- "Fiecare folosește propriile prompturi"
- Nu înțelege de ce consistența contează pentru o echipă

**Follow-up probe:** *"How do you measure if your instruction docs actually improved code quality?"*

**Răspunsul tău:**
> "Proxy metrics: (1) câte comentarii de code review cer ajustarea stilului/pattern-urilor (ar trebui să scadă). (2) Câte PR-uri ating code que generation rules direct (traci în .cursorrules/CLAUDE.md). (3) Time-to-merge pe PR-uri cu cod AI-generated (ar trebui să scadă pe măsură ce docs îmbunătățesc). Qualitative: 1:1 cu developerii — 'ai primit cod AI care nu respecta pattern-urile noastre?' Dacă da, actualizezi docs. E un feedback loop, nu o metrică exactă."

---

### Q14 — Refactor funcție de 200 linii generată de AI

**Întrebarea exactă:** "Your AI agent generated a 200-line function that works. But it violates your team's architecture patterns. Do you refactor manually or ask AI to refactor? Walk me through the conversation you'd have with the AI."

**✅ Strong Signals:**
- Folosești AI pentru refactoring dar cu instrucțiuni FOARTE specifice: "Split asta în X, Y, Z urmând pattern-ul nostru [specific]. Ține testele verzi."
- Rulezi codul refactorizat prin tests
- Compari diff
- Faci în pași, nu un prompt mare
- Știe când manual e mai rapid (schimbări mici, evidente)

**❌ Red Flags:**
- "L-aș shipa dacă merge"
- SAU refactorizează mereu manual pentru că nu are încredere în AI (fără middle ground)

**Follow-up probe:** *"When do you choose to NOT use AI and just write the code yourself?"*

**Răspunsul tău:**
> "Câteva scenarii: (1) Context critic de business pe care AI-ul nu-l cunoaște și explicarea lui ia mai mult decât scrierea codului. (2) Schimbări de 3-5 linii evidente — overhead-ul de prompt > scrierea directă. (3) Debugging de race conditions sau timing issues — AI-ul nu poate reproduce comportamentul; eu trebuie să urmăresc execution flow manual. (4) Code review-uri — prefer să înțeleg codul altcuiva fără AI ca filtru, pentru că vreau să prind probleme pe care AI-ul le poate valida greșit."

---

### Q15 — Un bug real pe care l-ai debugged cu AI

**Întrebarea exactă:** "Describe a bug you debugged recently using AI. What was the bug, what prompts did you use, and where did AI help vs where did it mislead you?"

**✅ Strong Signals:**
- Povestește cu specifice reale
- Știe să dea context la AI (error messages, stack traces, cod relevant)
- Identifică unde AI a hallicinat sau dat sfaturi greșite
- Gândire critică: "AI a sugerat X dar eu știam din stack trace că Y era problema reală"
- Debugging conversation iterativă

**❌ Red Flags:**
- Nu poate da un exemplu specific
- Răspuns generic: "Pun eroarea și o rezolvă"
- Nu menționează că AI a greșit sau a avut nevoie de corecție

**Pregătește O poveste reală din experiența ta.**
La Adobe Express — orice bug de performance, rendering sau integrare cu AI services.
La Arobs/CLIVE — orice bug de date, concurrent users, integrare API.

**Template de răspuns:**
> "La [proiect], am avut un bug unde [ce se întâmpla]. Am dat AI-ului [stack trace/cod relevant]. AI-ul a sugerat [ce a sugerat] — am implementat și nu a rezolvat pentru că [de ce nu mergea]. Contextul pe care AI-ul nu-l avea era [ce lipsea]. Am rafinat cu [ce ai adăugat ca context] și a identificat [soluția reală]. Ce m-a surprins: AI-ul era confident greșit prima dată — asta e un reminder că trebuie să validezi mereu, nu să accepți prima sugestie."

---

## ARCHITECTURE — Toate întrebările

---

### Q16 — PM spune "build feature X, might pivot in 2 weeks"

**✅ Strong Signals:** Feature flags din prima zi. Thin interface/adapter layers. Nu abstractizezi prematur — dar IZOLEZI. AI service calls în spatele unui interface simplu. Testezi contractul, nu implementarea. Fișiere scurte, boundaries clare.

**❌ Red Flags:** "Construiesc perfect de la început" SAU "Hack-uiesc și refactorizez mai târziu" fără middle ground. Fără izolare sau feature flags.

**Follow-up:** *"Give me an example of premature abstraction that burned you."*

---

### Q17 — 3 MVPs în paralel, fiecare cu AI diferit, shared auth + UI

**✅ Strong Signals:** Monorepo cu shared packages (auth, UI, config). Fiecare MVP e app independentă. Shared AI client cu adapter pattern per provider. Turborepo/Nx pentru build orchestration. Dependency graph clar — MVPs depind de shared, NICIODATĂ unele pe altele.

**❌ Red Flags:** 3 repo-uri separate cu cod copy-pastat. SAU o mega-app cu totul cuplat. Nu poate explica trade-offs mono vs multi-repo.

**Follow-up:** *"What do you put in the shared package vs what stays in the MVP?"*

**Răspunsul tău:**
> "În shared: auth logic (tokens, middleware), UI component library (Button, Input, Modal, Layout), config schemas (zod env validation), API client base (http client cu interceptors), types comune (User, ApiResponse). NU în shared: business logic specifică MVP-ului, AI prompts (fiecare MVP are use case diferit), route handlers, feature-specific components. Regula: dacă 2+ MVPs au nevoie de același lucru și ar fi necesar să îl schimbi simultan în ambele → merge în shared."

---

## DEEP TECHNICAL — Toate întrebările

---

### Q18 — Idempotency pentru duplicate jobs

**Întrebarea exactă:** "An HTTP request to your AI endpoint times out after 30s but the AI is still processing. The client retries, creating a duplicate job. How do you prevent this and ensure exactly-once processing?"

**✅ Strong Signals:**
- Idempotency key pe request (client generează UUID, trimite în header)
- Server verifică dacă job cu acel key există înainte să creeze unul nou
- Dacă există → returnezi existing job ID
- Client side: nu retry orbesc, verifici job status mai întâi
- Backend: job state machine previne double processing

**❌ Red Flags:** "Măresc timeout-ul." "Retry până merge." Fără concept de idempotency.

**Follow-up:** *"What happens if two identical requests arrive at the exact same millisecond?"*

**Răspunsul tău:**
> "Race condition. Fix: database-level unique constraint pe idempotency key + upsert atomic. Sau: Redis `SET key value NX EX 300` (set only if not exists) ca distributed lock înainte de a crea job-ul. Dacă două requesturi ajung simultan, unul câștigă lock-ul, celălalt primește 'job already exists'. Fără lock distributed, două requesturi simultane pot trece ambele verificarea 'există?' și crea ambele job-uri."

---

### Q19 — Stream AI token-by-token AND save to DB

**Întrebarea exactă:** "You need to stream AI responses token-by-token to the browser. The response also needs to be saved to a database once complete. Design the data flow."

**✅ Strong Signals:**
- AI SDK returnează ReadableStream
- **Tee the stream**: o ramură merge la SSE/WebSocket pentru client, alta bufferează pentru DB write
- SAU: stream la client ȘI acumulezi în memorie, scrii în DB la stream end
- Error handling: dacă stream se rupe la mijloc, salvezi rezultatul parțial cu flag 'incomplete'
- Client side: progressive rendering, handle reconnection

**❌ Red Flags:** Așteaptă răspunsul complet și trimite tot odată. Nu înțelege Node.js streams sau ReadableStream API.

**Follow-up:** *"How do you handle backpressure if the client reads slower than the AI produces tokens?"*

**Răspunsul tău:**
> "Node.js streams au backpressure built-in — dacă downstream (client) nu poate citi, `write()` returnează `false` și trebuie să aștepți evenimentul 'drain'. Cu SSE: dacă client-ul e lent, bufferul TCP se umple și eventual socket-ul devine 'not writable'. Soluție: monitorizezi `res.writableHighWaterMark` și `res.writableLength` — dacă buffer-ul e plin, pui upstream stream în pause. Alternativ: folosești `pipeline()` sau `pipe()` care gestionează backpressure automat."

---

### Q20 — Abstracție pentru 3 AI providers

**Întrebarea exactă:** "Your app uses 3 different AI providers (OpenAI, Anthropic, local Ollama). How do you design the integration layer so switching or adding providers doesn't require touching business logic?"

**✅ Strong Signals:**
- Adapter/Strategy pattern
- Common interface: `generateCompletion(prompt, options) → AsyncIterable<Token>`
- Fiecare provider implementează interfața
- Configuration determină ce adapter se încarcă
- AI SDK (Vercel) care face asta deja, sau custom abstraction
- Feature flags per provider. Fallback chain: primary → secondary → local
- Unified error types

**❌ Red Flags:** Hardcodezi provider-specific code în toată aplicația. "Refactorizăm când adăugăm un provider nou." Fără abstraction layer.

**Follow-up:** *"Where does prompt formatting live — in the adapter or in the business logic?"*

**Răspunsul tău:**
> "Prompt formatting trăiește în business logic (sau un prompt service separat), NU în adapter. Adapterul știe 'cum să apeleze' provider-ul, NU 'ce să ceară'. Business logic știe 'ce vrea să obțină'. Excepție: formatarea specifică provider-ului — de exemplu, Anthropic cere `Human:` prefix în prompt-uri mai vechi sau diferite structuri de messages. Asta e 'adapter concern' — translatezi din formatul tău intern în formatul specific provider-ului."

---

## Cheat Sheet — Ce să NU spui niciodată

```
❌ "Mut totul pe SSR"            → Always trade-offs
❌ "Adaug React.memo peste tot" → Only where it helps
❌ "Măresc timeout-ul"          → Async patterns
❌ "Retry până merge"           → Idempotency
❌ "Dacă trece testul, e bine"  → Manual review checklist
❌ "Folosesc ChatGPT uneori"    → Specific tools + real workflow
❌ "Fiecare cu prompturile lui" → Team AI instruction docs
❌ "Refactorizăm mai târziu"    → Isolate from day one
❌ "Nu testez componente complexe" → Mock at boundaries
❌ "Adaug mai multă RAM"        → Understand the actual problem
```

---

*[← 12 - Coding Assignments](./12-Coding-Assignments.md) | [← Ghid principal](./00-Ghid-Pregatire.md)*
