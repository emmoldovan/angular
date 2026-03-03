# 08 - Întrebări & Răspunsuri pentru Interviu

> Cele mai frecvente întrebări la un interviu TypeScript/Node.js cu focus pe
> agentic coding. Exersează-le cu voce tare!

---

## TypeScript

**Q: Care e diferența dintre `any`, `unknown` și `never`?**
> `any` dezactivează type checking — evit-o. `unknown` e type-safe `any` — trebuie să fac
> narrowing înainte să-l folosesc (type guard, assertion). `never` e tipul pentru cod care
> nu returnează niciodată (funcție care aruncă mereu eroare, loop infinit, ramură imposibilă).
> Useful pentru exhaustive checks în switch.

**Q: Ce sunt generics și de ce le folosești?**
> Generics permit scrirea codului refolosibil care funcționează cu mai multe tipuri, menținând
> type safety. Fără generics: sau scrii cod duplicat pentru fiecare tip, sau folosești `any`
> și pierzi tipurile. Exemple: `Array<T>`, `Promise<T>`, funcții utilitare, repository-uri.

**Q: Explică `Partial<T>`, `Required<T>`, `Pick<T, K>`, `Omit<T, K>`.**
> Utility types built-in: `Partial` face toate prop-urile opționale (util pentru update DTOs).
> `Required` le face obligatorii. `Pick` selectează un subset de chei. `Omit` exclude chei.
> De exemplu: `CreateUserDto = Omit<User, 'id' | 'createdAt'>`, `UpdateUserDto = Partial<CreateUserDto>`.

**Q: Ce e `discriminated union` și când îl folosești?**
> Union type cu un discriminator comun (property cu valoare literală unică per variant).
> TypeScript poate face narrowing automat. Folosesc pentru: Result types
> (`{ success: true; data: T } | { success: false; error: string }`), Redux actions, Event payloads.
> Evit cast-uri cu `as` și prind toate case-urile la compile time.

**Q: Cum faci module augmentation în TypeScript?**
> `declare global { namespace Express { interface Request { user?: User } } }` — extind
> tipurile existente fără să le modific. Util pentru a adăuga `req.user`, `process.env` type-safe,
> etc. Trebuie importat sau în un fișier `.d.ts` inclus în tsconfig.

---

## JavaScript

**Q: Explică event loop-ul.**
> Call stack execută cod sincron. Când e gol, event loop verifică: mai întâi microtask queue
> (Promise.then, queueMicrotask, process.nextTick) — se golește COMPLET. Apoi ia O singură
> macrotask (setTimeout, setInterval, I/O). Procesul se repetă. Asta înseamnă că Promises
> se rezolvă înainte de setTimeout, chiar cu delay 0.

**Q: Ce e closure și dă un exemplu real.**
> O funcție care capturează variabilele din scope-ul exterior (lexical environment) în care
> a fost creată. Exemplu real: factory functions, module pattern, event handlers cu state,
> debounce/throttle. Bug comun: `var` în for loop — closure capturează referința la `i`,
> nu valoarea. Fix: `let` (block scope) sau IIFE.

**Q: `==` vs `===` — de ce contează?**
> `===` strict equality (tip + valoare). `==` loose equality cu type coercion:
> `null == undefined` (true), `0 == false` (true), `'' == false` (true) — comportament
> counterintuitive. Folosesc ÎNTOTDEAUNA `===`. Excepție: `x == null` prinde și `undefined`.

**Q: Promisele sunt lazy sau eager?**
> Eager — executorul din `new Promise(executor)` rulează IMEDIAT (sincron). Promisele nu
> sunt lazy. Alternativa lazy: thunk (funcție care returnează promise), Observable (RxJS),
> sau generator/async generator.

**Q: `async/await` vs `.then()` — care preferi și de ce?**
> Prefer `async/await` pentru lizibilitate: codul pare sincron, error handling cu `try/catch`
> natural. `.then()` e util pentru operații paralele (`Promise.all`) și când nu vreau să
> blocheze (fire & forget). Uneori combin: `await Promise.all([asyncOp1(), asyncOp2()])`.

**Q: Ce e `prototype` și cum funcționează inheritance în JS?**
> Fiecare obiect JS are un prototype intern. Când accesezi o proprietate care nu există pe obiect,
> JS o caută pe prototype, apoi pe prototype-ul prototype-ului, etc. `class` e syntactic sugar
> pentru prototype-based inheritance. `Object.create(proto)` creează un obiect cu prototype-ul dat.

---

## Node.js & Express

**Q: De ce Node.js e non-blocking deși e single-threaded?**
> Operațiunile I/O (DB, file, network) sunt delegate la libuv (thread pool sau OS-level async I/O).
> Thread-ul JS nu așteaptă — continuă cu alte request-uri. Când I/O e gata, callback-ul intră
> în queue. Node e bun pentru I/O-bound, slab pentru CPU-bound (blochează event loop-ul).

**Q: Cum scalezi un server Node.js?**
> (1) Cluster module — utilizezi toate CPU core-urile (un process per core). (2) Worker Threads
> pentru CPU-intensive tasks. (3) Multiple instanțe cu load balancer (Nginx, AWS ALB).
> (4) Microservii — scalezi independent fiecare componentă. (5) Cache-uri (Redis) pentru
> request-uri repetitive.

**Q: Ce e middleware în Express și care e ordinea importantă?**
> Middleware = funcție `(req, res, next)` care interceptează request-ul. Express e un pipeline
> liniar — middlewarele rulează în ordinea definită. Ordinea standard: cors → helmet → body-parser
> → logger → rate limiter → routes → 404 → error handler. Error handler TREBUIE să fie ultimul
> și să aibă 4 parametri `(err, req, res, next)`.

**Q: Cum gestionezi erorile async în Express?**
> Express 4 nu prinde automat async errors. Soluție: wrapper `asyncHandler(fn)` care wrappează
> async handler și propagă erorile la `next(err)`. Error handler global cu 4 parametri
> diferențiază erori operaționale (AppError cu statusCode) de bug-uri (500 + log).

**Q: JWT vs Sessions — când alegi ce?**
> JWT pentru API-uri stateless, distributed systems, mobile clients — nu trebuie shared session
> store. Sessions pentru aplicații server-rendered cu revocare imediată. Eu prefer JWT cu
> access token scurt (15min) + refresh token în DB (revocabil, 7 zile în httpOnly cookie).

**Q: Cum previi memory leaks în Node.js?**
> (1) Event listeners fără cleanup (`.removeListener()` sau `off()`). (2) Closures care
> captează referințe mari. (3) Global variables acumulate. (4) Timers neîntrerupte.
> Detectare: `--inspect` + Chrome DevTools heap snapshots, clinic.js, AppSignal.

---

## Design Patterns & Arhitectură

**Q: Explică Layered Architecture.**
> Controllers (HTTP layer) → Services (business logic) → Repositories (data access).
> Fiecare layer cunoaște doar layer-ul de dedesubt. Beneficii: testabilitate (mock repository
> în service tests), schimb de implementare fără să atingi alte layers, separation of concerns.

**Q: Dependency Injection — cum îl implementezi fără framework?**
> Constructor injection: clasa primește dependențele ca parametri. Un Composition Root
> (de obicei `container.ts`) construiește tot arborele de dependențe la startup.
> Beneficiu major: testabilitate — injectez mocks în loc de implementări reale.

**Q: SOLID principles — dă un exemplu concret pentru fiecare.**
> S — Single Responsibility: `UserService` gestionează logica de users, nu trimite email-uri direct
> (delega la `EmailService`). O — Open/Closed: adaug un nou payment provider implementând
> `PaymentStrategy`, nu modific `PaymentService`. L — Liskov: orice `UserRepository` (Prisma,
> InMemory) poate înlocui interfața fără să strici codul. I — Interface Segregation: `ReadRepository`
> separat de `WriteRepository` pentru query/command separation. D — Dependency Inversion:
> `UserService` depinde de `IUserRepository` (interfață), nu de `PrismaUserRepository` (concretă).

**Q: Când folosești event-driven architecture?**
> Când am efecte secundare multiple la un eveniment (email + audit + analytics la user.created).
> Evit coupling-ul — fiecare handler e independent. Trade-off: mai greu de debugat (flow non-linear),
> erori în handlers pot fi pierdute dacă nu gestionez corect. Pentru async: job queues (Bull) sau
> message brokers (RabbitMQ, Kafka) pentru reliability.

---

## Agentic Coding

**Q: Cum folosești AI în workflow-ul tău de development?**
> Există două laturi pentru mine. Prima: la Adobe am construit produse unde AI-ul e livrat
> utilizatorilor — UI pentru generare de imagini cu Adobe Firefly. Am văzut din interior ce
> înseamnă să construiești un produs AI bun: stări edge, latență, UX pentru output nedeterminist.
> A doua latură: în workflow-ul meu propriu, folosesc Claude Code și Copilot — generez structura
> inițială, revizuiesc critic, iterez cu feedback specific. Nu accept output blackbox.
> Deciziile arhitecturale le iau eu, AI-ul accelerează execuția.

**Q: Ce faci când codul generat de AI nu e corect?**
> Nu re-promptez cu "încearcă din nou". Identific exact ce e greșit și de ce, și dau
> feedback specific: "findById trebuie să returneze null, nu să arunce eroare — las controller-ul
> să decidă ce face cu absența". Dacă problema e de context lipsă, adaug mai mult: cod existent,
> convenții, constrângeri de business. De obicei în 2-3 iterații ajung la ceva solid.

**Q: Cum verifici calitatea codului generat de AI?**
> Citesc codul rând cu rând — nu exist "blackbox acceptat". Specific: verific securitate
> (auth bypass, IDOR — orice user poate accesa resurse altui user?), edge cases (null, array gol,
> concurrență), și compatibilitate cu arhitectura existentă. La Adobe am dezvoltat un reflex
> puternic pentru performance — îmi uit automat la bundle impact și rendering cost.

**Q: Ai experiență cu MCP sau AI agents?**
> Conceptual, înțeleg arhitectura: un agent AI cu acces la tools externe (filesystem, bash,
> browser, API-uri) prin protocolul MCP. Lucrez cu Claude Code care e exact asta — vede
> fișierele mele, rulează comenzi, iterează. E diferit față de un chat — ai un agent care
> poate executa, nu doar sugera. Înțeleg de ce e util pentru agentic coding: poți da AI-ului
> context real (schema DB, codul existent, output-ul testelor) nu doar text.

---

## Întrebări pe care SĂ LE PUI TU

La finalul interviului, întreabă:

1. **"Care e cea mai mare provocare tehnică pe care o aveți în proiect acum?"**
   — Arată interes real, îți dă context pentru follow-up

2. **"Cum arată un ciclu tipic de development? Code review, testing, deployment?"**
   — Înțelegi cultura și procesele echipei

3. **"Cum folosiți AI-ul în echipă acum? Ce tools? Care sunt regulile?"**
   — Arată că ești aliniat cu direcția lor

4. **"Ce înseamnă succesul în primele 3 luni în acest rol?"**
   — Clarifici așteptările, arăți că gândești la impact

5. **"Cum e structura echipei și cu cine voi colabora cel mai mult?"**
   — Context social și tehnic

---

## Pitch personal — 60 secunde

> "Am peste 10 ani de experiență în frontend, și ceea ce mă diferențiază e că sunt
> framework agnostic — am lucrat în Angular, Vue, React, Lit, Stencil, Ionic, în
> contexte complet diferite. Nu mă definesc prin tool, ci prin capacitatea de a
> livra la nivel înalt indiferent de stack.
>
> Ultima experiență a fost la Adobe, ca Senior Frontend Engineer pe Adobe Express —
> am lucrat direct pe features de AI image generation, am integrat componente Lit cu
> servicii Node.js, și am îmbunătățit semnificativ performanța aplicației: reducere
> bundle size, timpi de încărcare mai mici, calitate mai bună pe mobile pentru milioane
> de utilizatori. Node.js și TypeScript nu sunt pentru mine o noutate — le-am folosit
> în producție la Adobe.
>
> Am și background de full stack — .NET Core la mai multe proiecte, și experiență
> solidă de leadership: am condus echipe de developeri, am creat un program de
> internship de 6 luni, am mentorizat zeci de juniori.
>
> Ce mă atrage la această poziție în mod specific e componenta de agentic coding.
> Am folosit activ AI în workflow-ul de development și știu că diferența nu e
> 'știi să copiezi de la AI' — ci 'știi să dai context corect, să verifici output-ul
> și să iei deciziile arhitecturale pe care AI-ul nu le poate lua'. Mă interesează
> să construiesc lucruri unde această combinație contează.
>
> Sunt curios cum arată proiectul vostru acum și unde credeți că pot aduce valoare."

### De ce funcționează acest pitch

- **Adobe Express + AI** — deschide imediat conversația despre experiența ta directă cu AI features în producție la o companie mare
- **Framework agnostic** — elimină obiecția "dar nu știe Node.js/React" înainte să apară
- **Node.js în producție la Adobe** — nu e teorie, e experiență reală
- **Mentoring + leadership** — arăți că nu ești doar executor, ci contributor la nivel mai mare
- **Finalul cu întrebare** — nu e monolog, e deschidere de dialog

### Variante de răspuns pentru follow-up imediat

**Dacă întreabă: "Spune-mi mai mult despre Adobe Express"**
> "Am lucrat pe features de UI pentru generarea de imagini cu AI — practic interfața prin
> care utilizatorii interacționau cu serviciile Adobe Firefly. Stack-ul era Lit pentru
> componente web și Node.js pentru tooling și middleware. Am îmbunătățit semnificativ
> performanța: am refactorizat și modularizat aplicația, am redus bundle size-ul și
> timpii de load. Am lucrat cross-functional cu product, design și backend — release-uri
> regulate cu impact vizibil pe milioane de utilizatori."

**Dacă întreabă: "Cum ai folosit AI în development?"**
> "În mai multe moduri. La Adobe am lucrat direct pe features AI ca produs — construind
> UI-ul pentru generarea de imagini. Separat, am folosit AI tools (Claude Code, Copilot)
> pentru a accelera development-ul: generez structura inițială, revizuiesc critic,
> iterez cu feedback specific. Am observat că cei care folosesc AI bine nu sunt cei care
> 'lasă AI-ul să scrie totul' — ci cei care știu ce să ceară, cum să verifice și cum să
> integreze. Acesta e modul meu de lucru."

**Dacă întreabă: "Ai lucrat cu React?"**
> "Am lucrat cu React Native la Movalio pe o aplicație cu 2000+ utilizatori. React web
> îl știu conceptual solid — hooks, state management, patterns — și dat fiind că vin
> din Angular și Lit, componentele și lifecycle-ul nu sunt concepte noi pentru mine.
> Nu l-am folosit în producție la scară mare, dar mă simt confortabil să fac tranziția
> rapid, ca și cu orice alt framework."

---

*[← 07 - React](./07-React-Overview.md) | [← Ghid principal](./00-Ghid-Pregatire.md)*
