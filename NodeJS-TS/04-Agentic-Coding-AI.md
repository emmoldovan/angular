# 04 - Agentic Coding & AI în Dezvoltare

> Cum lucrezi eficient cu AI, cum prezinți asta în interviu și ce înseamnă
> "agentic coding" în practică pentru un proiect Node.js + React.

---

## Ce înseamnă "agentic coding" pentru acest job

Poziția implică folosirea intensivă a AI pentru a **multiplica productivitatea** unui developer. Asta nu înseamnă:
- ❌ "Copiez ce dă ChatGPT"
- ❌ "Nu știu codul, AI-ul scrie tot"
- ❌ "Am rulat prompt-ul o singură dată și am terminat"

Înseamnă:
- ✅ **Orchestrezi** soluții: definești problema, lași AI să genereze, tu verifici și rafinezi
- ✅ **Recunoști** când AI greșește (hallucinations, pattern-uri vechi, securitate slabă)
- ✅ **Iterezi** rapid cu AI: feedback loop scurt între prompt → output → review → refine
- ✅ **Cunoști limitele**: AI nu știe contextul tău de business, nu știe arhitectura ta
- ✅ **Integrezi** output-ul AI în codebase-ul existent fără să rupi nimic

---

## Tooling — ce știi să folosești

### Claude Code (ce folosești acum)

```bash
# Claude Code = agent care rulează în terminal, are acces la:
# - filesystem (citire/scriere fișiere)
# - bash (rulează comenzi)
# - browser (prin MCP)
# - acces la alte tools prin MCP

# Exemple de sarcini agentic:
# "Creează un CRUD complet pentru User cu TypeScript, Express, Prisma"
# "Refactorizează toate controllere-le să folosească asyncHandler"
# "Adaugă validare Zod la toate endpoint-urile"
# "Scrie teste pentru UserService cu vitest"
```

### GitHub Copilot

```typescript
// Copilot e cel mai bun la:
// - Completare cod în context (cunoaște fișierele din workspace)
// - Generare tests (pattern-ul e repetitiv)
// - Documentație/JSDoc
// - Boilerplate (CRUD, middleware, DTOs)

// Exemplu de prompt eficient în Copilot Chat:
// "Based on the UserService pattern in src/users/, create a similar
//  OrderService with the same error handling and validation approach"
```

### Cursor

```
Cursor = VS Code + AI deeply integrated
- Composer: multi-file edits cu context
- Chat: discuție despre cod specific
- @files, @codebase: dai context explicit
- Rules for AI: definești convenții de cod
```

### Model Context Protocol (MCP)

```typescript
// MCP = protocol pentru a conecta AI agents la external tools
// Exemple de MCP servers utile:
// - filesystem: accede la fișiere local
// - github: creează PRs, citește issues
// - postgres/sqlite: query baze de date direct
// - fetch: face HTTP requests
// - puppeteer/playwright: browser automation

// Un developer care înțelege MCP poate să:
// 1. Configureze AI agents cu contextul corect (DB schema, API docs)
// 2. Creeze tool-uri custom pentru workflow-ul specific al echipei
// 3. Automatizeze sarcini repetitive (generare migrări, seeding, docs)
```

---

## Cum prezinți folosirea AI în interviu

### Povești STAR bazate pe experiența ta reală

---

**Povestea 1 — Adobe Express: UI pentru AI image generation**

Aceasta e povestea ta cea mai puternică pentru acest job. Ancorează orice discuție despre AI aici.

> **Situație:** La Adobe Express, în echipa care lucra pe features de creativitate,
> am primit task-ul de a construi UI-ul pentru generarea de imagini cu AI (Adobe Firefly).
>
> **Task:** Trebuia să integrez un flux UX complex — utilizatorul introduce un prompt,
> vede un loading state, primește mai multe variante, poate selecta și edita. Totul
> trebuia să funcționeze performant pe mobile, unde Adobe Express are un procent mare
> de utilizatori.
>
> **Acțiune:** Am construit componentele în Lit, am integrat cu serviciile Node.js
> din backend pentru API calls, am gestionat stări complexe (loading, error, generation
> progress, preview). Simultan, am lucrat la refactorizarea și modularizarea aplicației
> — reducerea bundle size-ului și a timpilor de load.
>
> **Rezultat:** Features livrate cross-functional cu product și design, impact vizibil
> pe performanță (load times mai mici, bundle mai mic), și o experiență de a construi
> UI pentru un produs AI real la scară de milioane de utilizatori.

**Ce poți spune în interviu:**
> "La Adobe am construit efectiv interfețele prin care utilizatorii interacționau cu
> AI-ul — nu eram eu utilizatorul AI-ului, ci construiam produsul. Asta mi-a dat
> o perspectivă interesantă: am înțeles ce înseamnă UX bun pentru AI-generated content,
> ce stări edge trebuie gestionate, și cum echilibrezi viteza de generare cu experiența
> utilizatorului."

---

**Povestea 2 — Arobs / CLIVE: 90% reducere birocrație prin digitalizare**

Folosește-o când întreabă de impact la scară sau de decizii arhitecturale.

> **Situație:** Pe proiectul CLIVE la Arobs, compania gestiona toate procesele
> administrative pe hârtie sau în sisteme disparate. Birocrația consuma un timp
> imens din ziua de lucru a angajaților.
>
> **Task:** Să construim o aplicație web care digitalizează toate procesele.
> Conduceam o echipă de 4 developeri, lucrând direct cu echipa de produs.
>
> **Acțiune:** Am ales o arhitectură cu microservices și Angular libraries pentru
> scalabilitate. Am implementat un workflow Jira eficient pentru cerințe rapide.
> Am mentorizat cei 4 juniori în Angular, .NET Core și best practices de documentare.
>
> **Rezultat:** 90% reducere a timpului petrecut pe sarcini birocratice. Nu am măsurat
> un feature — am măsurat impactul real în viața de zi cu zi a oamenilor din companie.

**Ce poți spune în interviu:**
> "Ce am învățat pe CLIVE e că soluțiile tehnice bune nu sunt cele mai elegante —
> sunt cele care rezolvă problema reală. 90% reducere de birocrație nu a venit dintr-o
> arhitectură perfectă, ci din înțelegerea profundă a proceselor pe care le digitalizăm."

---

**Povestea 3 — Movalio / gosnippet: produs real cu 2000+ utilizatori**

Folosește-o dacă întreabă de experiența cu produse consumer-facing sau React.

> **Situație:** La Movalio, am lucrat pe gosnippet.com — o aplicație de tip Kindle/highlighting
> cu peste 2000 de utilizatori activi.
>
> **Task:** Adăugare de features noi: highlighting pe documente PDF, recrearea
> highlight-urilor live în browser, export în formate diferite.
>
> **Acțiune:** Am implementat în Vue, Vuetify, Vuex pentru web și React Native pentru
> mobile. A trebuit să înțeleg cum funcționează rendering-ul PDF în browser, cum
> persist highlight-urile cu precizie (coordonate relative, nu absolute).
>
> **Rezultat:** Features livrate pe un produs real cu utilizatori reali — am primit
> feedback direct de la utilizatori, ceea ce e altceva față de proiecte enterprise interne.

---

**Cum ancorezi experiența ta în agentic coding**

Dacă nu ai folosit Claude Code / Cursor în producție pe proiecte anterioare (e ok!), prezintă-l onest:

> "Agentic coding cu AI tools ca Claude Code e ceva pe care l-am explorat activ recent —
> inclusiv pentru pregătirea acestui interviu (am folosit Claude Code să genereze
> materiale de studiu, să refactorizeze cod, să scrie teste). La Adobe am trăit
> cealaltă față — am construit produse unde AI-ul e livrat utilizatorilor.
> Acum vreau să combin: să construiesc cu AI și pentru utilizatori care construiesc cu AI."

---

## Prompt Engineering pentru cod — pattern-uri eficiente

### Pattern 1: Context + Constraints + Example

```
CONTEXT: Am un Express API TypeScript cu Prisma ORM și Zod pentru validare.
Convențiile din proiect:
- Toate handlers sunt async și wrapped în asyncHandler()
- Erorile se aruncă cu AppError(message, statusCode, code)
- Validarea se face în middleware cu validate(schema)

TASK: Creează un endpoint PATCH /users/:id pentru update parțial.

CONSTRAINTS:
- Un user nu poate schimba email-ul altuia (verificare unicitate)
- Passwordul se updatează separat (nu în acest endpoint)
- Returnează 404 dacă user-ul nu există

EXAMPLE: Uită-te la src/routes/orders/orders.controller.ts pentru stilul corect.
```

### Pattern 2: Iterativ cu feedback specific

```
Iterația 1: "Generează UserService cu CRUD basic"
→ Revizuiești output-ul

Iterația 2: "Codul e bun, dar:
  - findById trebuie să returneze null, nu să arunce eroare (las controller-ul să decidă)
  - create trebuie să hash-uiască parola înainte de a salva
  - adaugă soft delete (deletedAt) în loc de delete hard"
→ Revizuiești din nou

Iterația 3: "Acum adaugă tipuri TypeScript pentru toate return types și
  adaugă JSDoc pentru metodele publice"
```

### Pattern 3: Test-first cu AI

```
"Înainte să generezi implementarea, scrie testele pentru UserService.
Testele trebuie să acopere:
1. findById: găsit, negăsit, deleted
2. create: succes, email duplicate, validare eșuată
3. update: succes, user inexistent, email duplicate

Folosește vitest și mock Prisma cu vi.mock('./prisma')"

→ Revizuiești testele (ușor de verificat logica)
→ Ceri implementarea care trece testele
```

---

## Ce știe AI-ul bine și ce NU știe

### AI-ul e excelent la:
- **Boilerplate** repetitiv: CRUD, DTOs, schemas, migration files
- **Patterns cunoscute**: middleware, repositories, services
- **Refactoring mecanic**: rename, extract method, format code
- **Tests**: unit tests pentru logică simplă
- **Documentație**: JSDoc, README, API docs
- **Bug debugging**: dacă îi dai contextul complet (error + stack + code)
- **Security checklist**: OWASP top 10 pentru cod existent

### AI-ul greșește la:
- **Contextul tău specific de business**: nu știe regulile tale
- **Arhitectura ta curentă**: poate propune incompatibilități
- **Date sensibile**: nu-i da credențiale, date de producție
- **Cod nou/versiuni recente**: hallucinations pentru API-uri noi
- **Trade-offs complexe**: nu înțelege costurile și constrângerile tale
- **Securitate subtilă**: IDOR, race conditions, timing attacks

---

## Agentic Coding Workflow — exemplu complet

```
Task: "Adaugă autentificare cu Google OAuth2 la API-ul nostru Express"

Pasul 1 — TU definești arhitectura:
  - Ce bibliotecă? (passport.js? google-auth-library? manual?)
  - Unde stocăm user-ii noi (DB schema?)
  - Ce returnăm? (JWT? session?)
  - Ce se întâmplă cu user-ii care au deja cont cu email/password?

Pasul 2 — Ceri AI contextul:
  "Explică-mi flow-ul OAuth2 Authorization Code cu PKCE și
   ce endpoints trebuie să implementez pe server-ul Express"

Pasul 3 — AI generează structura:
  - Prisma migration pentru OAuthAccount
  - Callback route
  - Token exchange logic
  - User creation/linking logic

Pasul 4 — TU revizuiești:
  ✓ State parameter pentru CSRF protection? (AI-ul uneori uită)
  ✓ Token-urile Google sunt stocate? (nu vrei asta, de obicei)
  ✓ Error handling pentru "access denied" de user?
  ✓ Rate limiting pe callback?

Pasul 5 — Iterezi cu AI:
  "Lipsește validarea state parameter-ului pentru CSRF.
   Adaugă stocare temporară a state în Redis cu TTL de 10 minute"

Pasul 6 — TU testezi manual:
  - Fluxul complet în browser
  - Edge cases: user refuză permisiunile, token invalid

Pasul 7 — AI generează tests:
  "Scrie integration tests pentru /auth/google/callback acoperind:
   succes, state invalid, error de la Google, email duplicate"

Total: 2-3 ore vs 1-2 zile manual
```

---

## Cum răspunzi la întrebările despre AI în interviu

**Q: Cum folosești AI în workflow-ul tău zilnic?**
> "Folosesc Claude Code ca un pair programmer avansat. Definesc problema și arhitectura,
> generez soluții cu AI, revizuiesc critic (mai ales securitate și edge cases),
> iterez cu feedback specific. AI multiplică productivitatea pentru task-uri repetitive
> sau boilerplate, dar deciziile arhitecturale și business logic le fac eu."

**Q: Ce faci când AI-ul generează cod greșit?**
> "Feedback specific, nu 'e greșit'. Explic exact CE e greșit și DE CE, și cer să
> corecteze acel aspect. Dacă e o problemă fundamentală de înțelegere, îi dau mai mult
> context sau îl abordez diferit (ex: test-first, sau îl rog să explice abordarea
> înainte să genereze cod)."

**Q: Îți faci griji că AI-ul îți va lua job-ul?**
> "Nu în roluri de Senior+ în viitorul apropiat. Cineva trebuie să decidă CE să construiești,
> să revieweze că e corect, să înțeleagă implicațiile de securitate, să navigheze
> ambiguitatea cerințelor de business. AI-ul accelerează execuția, nu înlocuiește
> judecata inginerească."

**Q: Ai folosit MCP sau ai construit agenți AI?**
> "Da, am configurat MCP servers pentru [menționează ce ai configurat].
> Conceptual, am lucrat cu Claude Code care e el însuși un agent — vede filesystem-ul,
> rulează comenzi, iterează. Înțeleg arhitectura: tools definite, context window,
> orchestrare multi-step."

---

## Resurse pentru aprofundat

- **Anthropic docs**: [docs.anthropic.com](https://docs.anthropic.com) — Claude API, system prompts, tool use
- **MCP**: [modelcontextprotocol.io](https://modelcontextprotocol.io) — protocol spec și servers disponibile
- **Cursor rules**: `.cursorrules` sau `.cursor/rules/` — convenții pentru AI în proiect
- **GitHub Copilot Workspace**: agentic feature pentru issue → PR automat

---

*[← 03 - Node.js](./03-NodeJS-Express.md) | [05 - Patterns →](./05-Patterns-si-Arhitectura.md)*
