# 11. Leadership si Soft Skills — Angular Principal Engineer

## Cuprins

1. [Responsabilitatile unui Principal Engineer](#1-responsabilitatile-unui-principal-engineer)
2. [Mentoring si coaching echipe](#2-mentoring-si-coaching-echipe)
3. [Technical decision making (RFCs, ADRs)](#3-technical-decision-making-rfcs-adrs)
4. [Comunicarea deciziilor tehnice](#4-comunicarea-deciziilor-tehnice)
5. [Gestionarea conflictelor tehnice](#5-gestionarea-conflictelor-tehnice)
6. [Influenta fara autoritate directa](#6-influenta-fara-autoritate-directa)
7. [Code reviews eficiente](#7-code-reviews-eficiente)
8. [Driving technical vision](#8-driving-technical-vision)
9. [Intrebari comportamentale frecvente (STAR method)](#9-intrebari-comportamentale-frecvente-star-method)
10. [Cum te diferentiezi de un Senior Engineer](#10-cum-te-diferentiezi-de-un-senior-engineer)
11. [Intrebari frecvente de interviu](#intrebari-frecvente-de-interviu)

---

## 1. Responsabilitatile unui Principal Engineer

### Technical Vision and Strategy

Un Principal Engineer defineste directia tehnica la nivel de **organizatie**, nu doar la nivel de echipa. Aceasta inseamna:

- **Crearea unui Technology Radar** — evaluarea periodica a tehnologiilor folosite, adoptate, in proba sau abandonate
- **Definirea principiilor arhitecturale** — reguli care ghideaza toate echipele (ex: "services should own their data", "prefer composition over inheritance", "every API must be versioned")
- **Stabilirea standardelor tehnice** — coding conventions, testing strategy, deployment pipeline, security baseline
- **Planificarea pe termen lung** — anticiparea nevoilor tehnice ale business-ului cu 6-18 luni inainte

```
Exemplu concret:
"Am observat ca 4 echipe construiau propriile solutii de state management
in Angular. Am propus si implementat o strategie comuna bazata pe NgRx
cu o librarie shared de patterns, reducand duplicarea cu 60% si
timpul de onboarding al developerilor noi de la 3 saptamani la 1 saptamana."
```

### Cross-team Technical Alignment

Principal Engineer-ul este **liantul tehnic** intre echipe:

- Participa la design reviews ale tuturor echipelor
- Identifica oportunitati de reutilizare a codului si a pattern-urilor
- Mediaza conflicte tehnice intre echipe
- Se asigura ca deciziile locale nu afecteaza negativ sistemul global
- Mentine o **arhitectura coerenta** la nivel de organizatie

### Architecture Decisions at Org Level

La nivel organizational, deciziile arhitecturale au impact enorm:

| Decizie | Impact | Exemplu |
|---------|--------|---------|
| Alegerea framework-ului | 3-5 ani de mentenanta | Angular vs React pentru toata organizatia |
| Strategia de micro-frontends | Deployment, teams, DX | Module Federation vs Single SPA |
| Design system | UX consistency, viteza | Custom vs Material vs Tailwind |
| API strategy | Frontend-backend contract | REST vs GraphQL vs tRPC |
| Testing strategy | Quality, CI time | Unit-heavy vs E2E-heavy vs balanced |

### Identificarea si rezolvarea problemelor sistemice

Un Principal Engineer nu rezolva bug-uri izolate — identifica **pattern-uri de probleme**:

```
Gandire la nivel de Senior:
"Aceasta componenta are o problema de performance. O optimizez."

Gandire la nivel de Principal:
"Trei echipe au aceeasi problema de performance in componente similare.
Cauza root este lipsa unui pattern standard pentru virtual scrolling
si lazy loading in design system-ul nostru. Voi crea un RFC pentru
un set de componente performante reutilizabile si voi mentora o echipa
sa le construiasca."
```

**Probleme sistemice tipice:**
- Build times crescute exponential (solutie: Nx, incremental builds)
- Inconsistente in error handling intre echipe
- Lipsa observability (solutie: standardizarea logging si monitoring)
- Debt tehnic acumulat fara strategie de plata
- Onboarding lent al developerilor noi

### Raising the Technical Bar

**Metode concrete:**

1. **Engineering excellence programs** — workshop-uri lunare pe teme avansate
2. **Design review process** — orice feature semnificativa trece printr-un review
3. **Post-mortem culture** — invatam din incidente, fara blamarea
4. **Golden path documentation** — ghiduri "blessed" pentru scenarii comune
5. **Internal tech talks** — prezentari saptamanale de 30 min
6. **Code quality metrics** — tracking si vizibilitate, nu "policing"

### Building Technical Culture

Cultura tehnica se construieste prin **actiuni repetate**, nu prin declaratii:

- **Sarbatoresti refactoring-ul** la fel cum sarbatoresti feature-urile noi
- **Aloci timp explicit** pentru technical debt (ex: 20% din sprint)
- **Recunosti public** contributiile tehnice ale colegilor
- **Documentezi deciziile** — nu doar codul
- **Promovezi experimentarea** — hackathons, 20% time, PoC-uri
- **Modelezi comportamentul** — scrii teste, faci code reviews detaliate, documentezi

### Diferenta intre nivele de seniority (IC track)

| Aspect | Staff Engineer | Principal Engineer | Distinguished Engineer |
|--------|---------------|-------------------|----------------------|
| **Scope** | Echipa / 2-3 echipe | Organizatie / Departament | Companie / Industrie |
| **Orizont de timp** | 3-6 luni | 6-18 luni | 1-5 ani |
| **Influenta** | Directa, prin cod | Indirecta, prin altii | Vizionara, prin directie |
| **Cod** | 50-70% din timp | 20-40% din timp | 10-20% din timp |
| **Output principal** | Features complexe, mentoring local | Strategii tehnice, aliniere cross-team | Inovatie, thought leadership |
| **Decizii** | Intra-echipa | Cross-echipe | Cross-companie |
| **Comunicare** | Engineers | Engineers + Managers | Engineers + VPs + CTO |
| **Analogie** | Capitanul unei nave | Amiralul flotei | Cartograful oceanelor |

---

## 2. Mentoring si coaching echipe

### Mentoring vs Coaching — diferenta fundamentala

| Aspect | Mentoring | Coaching |
|--------|-----------|----------|
| **Directie** | Mentor-ul ofera raspunsuri si experienta | Coach-ul pune intrebari si ghideaza |
| **Focus** | Transfer de cunostinte | Dezvoltarea abilitatii de gandire |
| **Relatie** | "Am trecut prin asta, iata ce am invatat" | "Ce crezi ca ar fi cea mai buna abordare?" |
| **Cand** | Persoana e la inceput, lipseste contextul | Persoana are cunostinte dar blocaje |
| **Exemplu** | "Foloseste OnPush change detection pentru performance" | "Ce optiuni ai identificat pentru optimizarea performantei? Care sunt trade-off-urile?" |
| **Rezultat** | Stie ce sa faca | Stie cum sa gandeasca |

**Regula de aur:** Cu cat persoana e mai seniorita, cu atat folosesti **mai mult coaching** si **mai putin mentoring**.

### Structura unei sesiuni 1-on-1

```
Frecventa: Saptamanal, 30-45 minute
Format: Video call sau in persoana (NU pe chat)

STRUCTURA:
=========

[5 min] Check-in personal
  - Cum te simti? Cum a fost saptamana?
  - (Construiesti rapport si incredere)

[10 min] Progresul pe obiective
  - Ce ai realizat saptamana aceasta?
  - Ce ai invatat?
  - Ce te-a surprins?

[10 min] Blocaje si provocari
  - Ce te blocheaza acum?
  - (Aplica coaching: pune intrebari inainte de a da raspunsuri)
  - Ce ai incercat deja?
  - Ce optiuni vezi?

[10 min] Dezvoltare profesionala
  - Unde vrei sa ajungi in 6 luni?
  - Ce skill-uri iti lipsesc?
  - Ce oportunitate din proiect te-ar ajuta sa cresti?

[5 min] Action items
  - 1-2 lucruri concrete pana saptamana viitoare
  - (Persoana le defineste, nu mentorul)

SFATURI:
- Tine un document shared cu note din fiecare sesiune
- Urmareste progresul pe termen lung (nu doar saptamanal)
- Ajusteaza balanta mentoring/coaching in functie de subiect
- Sarbatoreste progresul, nu doar identifica gap-urile
```

### Growing Senior Engineers to Staff Level

Tranzitia de la Senior la Staff este una dintre cele mai dificile din cariera. Iata cum ajuti:

**1. Extinde scope-ul treptat:**
```
Luna 1-2: Mentee conduce design review-uri in echipa
Luna 3-4: Mentee propune un RFC minor cross-team
Luna 5-6: Mentee mentoreaza un mid-level engineer
Luna 7-8: Mentee conduce o initiativa tehnica cross-team
Luna 9-12: Mentee defineste si implementeaza o strategie tehnica
```

**2. Dezvolta skill-urile care lipsesc de obicei:**

| Skill Senior | Skill Staff (de dezvoltat) |
|-------------|---------------------------|
| Scrie cod excelent | Defineste cum altii scriu cod (standards) |
| Rezolva probleme complexe | Identifica proactiv problemele |
| Comunica cu echipa | Comunica cross-team si cu management |
| Implementeaza design-uri | Creeaza design-uri si le valideaza |
| Face code reviews | Defineste procesul de code review |
| Intelege business-ul | Influenteaza roadmap-ul tehnic |

**3. Ofera oportunitati de vizibilitate:**
- Prezentari in tech talks interne
- Scrie ADR-uri si RFC-uri
- Reprezentarea echipei in guild-uri sau working groups
- Participarea la interviuri tehnice (ca interviewer)
- Ownership pe un proiect cross-team

### Creating Learning Paths

```
Exemplu: Learning Path pentru Angular Engineer (Mid -> Senior)

FUNDAMENTALS MASTERY (Luna 1-3)
================================
[ ] RxJS advanced patterns (switchMap, exhaustMap, custom operators)
[ ] Angular change detection internals (Zone.js, OnPush, signals)
[ ] TypeScript advanced types (conditional, mapped, template literal)
[ ] NgRx / Signal Store deep dive
    Verificare: Code review de un proiect personal
    Milestone: Poate explica trade-off-urile fiecarui pattern

ARCHITECTURE (Luna 3-6)
========================
[ ] Design patterns in Angular (facade, adapter, mediator)
[ ] Micro-frontend architecture
[ ] Monorepo management cu Nx
[ ] Performance optimization (bundle size, runtime, LCP, CLS)
    Verificare: Propune si implementeaza un refactoring major
    Milestone: Conduce un design review

LEADERSHIP (Luna 6-9)
======================
[ ] Scrie 2 ADR-uri
[ ] Mentoreaza un junior engineer
[ ] Prezinta un tech talk intern
[ ] Participa activ in 5 code reviews pe saptamana
    Verificare: Feedback 360 de la colegi
    Milestone: Recunoscut ca go-to person pe un domeniu

IMPACT (Luna 9-12)
===================
[ ] Conduce o initiativa cross-team
[ ] Defineste un standard tehnic adoptat de organizatie
[ ] Contribuie la un proiect open-source sau creaza un tool intern
    Verificare: Impact masurabil (metrici)
    Milestone: Ready for Senior promotion
```

### Pair Programming ca instrument de mentoring

**Cand sa folosesti pair programming:**
- Cod complex unde mentee-ul se blocheaza
- Introducerea unui pattern nou
- Debugging dificil (mentee-ul invata procesul de gandire)
- Code review live (mai eficient decat async)

**Cum sa faci pair programming eficient ca mentor:**

```
DRIVER/NAVIGATOR MODEL (recomandat):

Scenariu 1: Mentee e Driver (tastatura)
  - Mentee scrie cod
  - Mentorul intreaba: "De ce ai ales aceasta abordare?"
  - Mentorul sugereaza alternative: "Ce s-ar intampla daca..."
  - Beneficiu: Mentee-ul gandeste activ

Scenariu 2: Mentorul e Driver (demonstratie)
  - Mentorul scrie cod si gandeste cu voce tare
  - "Aleg sa fac asta pentru ca..."
  - "Alternativa ar fi fost... dar nu o aleg pentru ca..."
  - Beneficiu: Mentee-ul vede procesul de gandire

REGULA: 70% din timp mentee-ul e Driver.
```

### Building a Culture of Learning

**Practici concrete:**

1. **Tech Talk Tuesday** — prezentare de 30 min saptamanala, rotatie intre membri echipei
2. **Book Club** — o carte tehnica pe luna, discutie in grup
3. **Failure Friday** — post-mortem-uri fara blame, focus pe invatare
4. **Exploration Sprints** — un sprint pe trimestru dedicat experimentarii
5. **Internal Blog** — articole scrise de echipa, vizibile in organizatie
6. **Conference Budget** — alocarea bugetului pentru conferinte si cursuri
7. **Shadowing** — un developer petrece o zi cu o alta echipa

### Exemplu practic: Cum ajuti un mid-level engineer sa creasca

```
CONTEXT: Maria, mid-level Angular developer, 2 ani experienta.
Puncte forte: scrie cod curat, buna la CSS, fiabila pe deadline-uri.
De imbunatatit: nu scrie teste, evita problemele de arhitectura,
nu participa in code reviews.

PLAN PE 6 LUNI:
================

Luna 1 — Stabilirea bazei
  - 1-on-1 initial: intelege obiectivele Mariei (vrea sa devina Senior)
  - Definim impreuna 3 obiective SMART
  - Pair programming saptamanal (1h) pe testing
  - Task: Maria scrie teste pentru o componenta existenta

Luna 2 — Testing confidence
  - Maria scrie teste pentru fiecare PR al ei (cerinta)
  - Review-ul meu se concentreaza pe calitatea testelor
  - Introduc conceptul de TDD pe un feature mic
  - Task: Maria prezinta "Testing Best Practices" in echipa

Luna 3 — Gandire arhitecturala
  - Maria participa la design review-uri (initial doar observa)
  - Ii dau sa citeasca si sa comenteze un ADR
  - Discutam design patterns in Angular (1-on-1)
  - Task: Maria propune arhitectura pentru un feature mediu

Luna 4 — Code review skills
  - Maria face minimum 3 code reviews pe saptamana
  - Discut cu ea calitatea feedback-ului dat
  - Ii arat cum dau eu feedback constructiv
  - Task: Maria identifica un pattern problematic in codebase

Luna 5 — Ownership
  - Maria conduce un refactoring mic (cu suport)
  - Participa la planning tehnic
  - Scrie primul ADR (cu review-ul meu)
  - Task: Maria mentoreaza un junior pe un task specific

Luna 6 — Evaluare si next steps
  - Review complet al progresului
  - Feedback 360 de la echipa
  - Plan pentru urmatoarele 6 luni
  - Discutie despre timeline de promovare
```

---

## 3. Technical Decision Making (RFCs, ADRs)

### RFC (Request for Comments) Process

Un RFC este un document formal prin care propui o schimbare tehnica semnificativa si soliciti feedback de la stakeholders.

**Procesul complet:**

```
1. IDENTIFICARE (1-2 zile)
   Autor: Identifica problema si solutia propusa
   Output: Draft initial

2. DRAFT (2-5 zile)
   Autor: Scrie RFC-ul complet
   Include: Context, propunere, alternative, impact
   Reviewers: 2-3 persoane de incredere (pre-review)

3. REVIEW (5-10 zile)
   Distribuit: Tuturor stakeholders
   Format: Comentarii async pe document
   Autor: Raspunde la comentarii, itereaza

4. DISCUSSION (1-2 ore)
   Format: Meeting dedicat (daca e necesar)
   Focus: Rezolvarea dezacordurilor ramase
   Regula: Fiecare persoana isi exprima pozitia

5. DECIZIE (1 zi)
   Decision maker: Autorul sau un delegat
   Options: Accept / Accept with changes / Reject / Defer
   Documentat: Decizia finala si rationale-ul

6. IMPLEMENTARE
   Plan: Break down in tasks
   Tracking: In project management tool
   Review: La finalul implementarii vs RFC
```

**Template RFC:**

```markdown
# RFC-042: Migrarea la Angular Signals si eliminarea Zone.js

**Autor:** [Nume]
**Data:** [Data]
**Status:** Draft | In Review | Accepted | Rejected | Superseded
**Reviewers:** [Lista]
**Decision deadline:** [Data]

## Summary
O propozitie clara de 2-3 randuri despre ce propunem.

## Motivation
De ce facem aceasta schimbare? Ce problema rezolvam?
Ce se intampla daca NU facem nimic?

## Detailed Design
### Arhitectura propusa
[Diagrame, cod exemplu, flow-uri]

### Migration Strategy
[Cum trecem de la starea actuala la cea propusa]

### Phases
Phase 1: [descriere] — estimare: 2 sprints
Phase 2: [descriere] — estimare: 3 sprints
Phase 3: [descriere] — estimare: 1 sprint

## Alternatives Considered
### Alternative A: [Nume]
- Pro: ...
- Con: ...
- De ce nu: ...

### Alternative B: [Nume]
- Pro: ...
- Con: ...
- De ce nu: ...

## Impact Analysis
### Performance: [descriere impact]
### Developer Experience: [descriere impact]
### Migration effort: [estimare]
### Risk: [riscuri identificate si mitigari]

## Open Questions
1. [Intrebare nerezolvata]
2. [Intrebare nerezolvata]

## References
- [Link-uri relevante]
```

### ADR (Architecture Decision Record)

Un ADR este un document scurt care captureaza **o singura decizie arhitecturala** si contextul ei.

**De ce ADR-uri?**
- Noii membri ai echipei inteleg **de ce** s-a ales o solutie
- Evitam reluarea deciziilor deja luate
- Documentam ce am luat in considerare si ce am respins
- Cream un "audit trail" al gandirilor tehnice

**Format standard (Michael Nygard):**

```markdown
# ADR-017: Adoptarea NgRx Signal Store pentru state management

**Data:** 2026-01-15
**Status:** Accepted

## Context

Aplicatia noastra Angular foloseste in prezent un mix de:
- Servicii cu BehaviorSubject (echipa Payments — 12 servicii)
- NgRx Store clasic (echipa Users — 3 feature states)
- State local in componente (echipa Dashboard — ~40 componente)

Aceasta inconsistenta cauzeaza:
- Timp de onboarding crescut (developerii noi trebuie sa invete 3 abordari)
- Imposibilitatea de a partaja state management patterns intre echipe
- Debugging dificil (nu exista un singur tool — DevTools functioneaza
  doar pentru NgRx Store)
- Performance issues: serviciile cu BehaviorSubject nu beneficiaza de
  OnPush change detection in mod predictibil

## Decision

Adoptam **NgRx Signal Store** ca solutie standard de state management
pentru toate echipele, cu urmatoarele reguli:
1. Feature-urile noi folosesc exclusiv Signal Store
2. Codul existent se migreaza oportunistic (la refactoring sau bug fix)
3. Deadline de migrare completa: Q3 2026

## Consequences

### Pozitive
- Un singur pattern de state management in toata organizatia
- Signal Store se integreaza nativ cu Angular Signals (viitorul framework-ului)
- Bundle size mai mic decat NgRx Store clasic (~2KB vs ~12KB)
- DevTools support prin @ngrx/store-devtools
- API mai simplu, curba de invatare mai mica

### Negative
- Efort de migrare estimat: ~120 ore (distribuit pe 3 echipe)
- Echipa Users pierde investitia in NgRx effects complexe
- Signal Store e relativ nou — posibil sa intampine limitari
  pe edge case-uri pe care nu le-am anticipat

### Riscuri
- Signal Store nu suporta inca toate feature-urile NgRx Store
  (ex: router state integration) — mitigare: verificam inainte
  de migrarea fiecarui feature
- Rezistenta echipelor la schimbare — mitigare: workshop-uri si
  pair programming sessions

## Alternatives Considered

### 1. Standardizare pe NgRx Store clasic
Respins: overhead prea mare, direction-ul Angular e catre signals

### 2. Standardizare pe servicii cu BehaviorSubject
Respins: lipsesc DevTools, nu se integreaza cu signals, boilerplate

### 3. Asteptare pana signals e mai matur
Respins: inconsistenta actuala are deja cost real, signals e stabil
de la Angular 17
```

### Cand sa scrii un RFC vs cand sa decizi direct

| Scenariu | Decizie directa | RFC necesar |
|----------|-----------------|-------------|
| Redenumire variabila | Da | Nu |
| Adaugare librarie utilitara mica | Da | Nu |
| Schimbarea state management | Nu | Da |
| Alegerea unui pattern de testing | Depinde* | Depinde* |
| Migrarea la o noua versiune Angular | Nu | Da |
| Crearea unui design system | Nu | Da |
| Adaugarea unui nou endpoint API | Da | Nu |
| Schimbarea strategiei de deployment | Nu | Da |
| Introducerea micro-frontends | Nu | Da (RFC mare) |

*Depinde de impactul pe organizatie. Daca afecteaza o echipa: decizie directa. Daca afecteaza mai multe: RFC.

**Regula generala:** Daca decizia afecteaza **mai mult de o echipa** sau **nu e usor reversibila**, scrie un RFC.

### Getting Buy-in from Stakeholders

**Strategia de buy-in in 5 pasi:**

```
1. PRE-ALIGNMENT (inainte de RFC)
   - Vorbeste individual cu stakeholders cheie
   - Intelege preocuparile lor
   - Incorporeaza feedback-ul in draft
   - "Am discutat cu X si Y si am incorporat feedback-ul lor"

2. PROBLEM FIRST, SOLUTION SECOND
   - Incepe cu durerea (cost actual al problemei)
   - "Pierdem 3 ore/saptamana/developer din cauza build times"
   - Abia apoi propune solutia
   - Oamenii trebuie sa simta problema inainte sa accepte solutia

3. DATA, NOT OPINIONS
   - Benchmark-uri: "Build time actual: 8 min, cu Nx: 2 min"
   - Metrici: "Reducere bundle size cu 40%"
   - Cost: "Estimam 2 sprints de efort, ROI in 3 luni"
   - Exemple: "Echipa X a facut asta si a obtinut Y"

4. ACKNOWLEDGE TRADE-OFFS
   - "Da, asta va costa 2 sprints de efort"
   - "Da, exista riscul ca..."
   - "Am considerat alternativa X dar..."
   - Credibilitatea vine din onestitate, nu din optimism

5. MAKE IT EASY TO SAY YES
   - Plan de implementare clar si incremental
   - Rollback strategy daca nu functioneaza
   - Quick win initial (demonstreaza valoare rapid)
   - Ofera-te sa conduci implementarea
```

---

## 4. Comunicarea deciziilor tehnice

### Adaptarea mesajului la audienta

Aceeasi decizie trebuie comunicata diferit in functie de audienta:

**Exemplu: Decizia de a migra de la Angular Universal la Angular SSR cu hydration**

**Catre Engineers:**
```
Migram de la Angular Universal la noul Angular SSR cu partial hydration.
Motivele tehnice:
- Hydration selectiva reduce TTI cu ~40% (benchmark-urile noastre:
  de la 3.2s la 1.9s)
- Event replay elimina flickering-ul pe interactiuni early
- transferState devine automat (nu mai trebuie TransferHttpCacheModule)
- Aliniere cu roadmap-ul Angular (Universal e deprecated)

Impact pe echipe:
- Echipa A: 3 componente de migrat (estimare: 2 zile)
- Echipa B: Server module de actualizat (estimare: 1 zi)
- Echipa C: E2E tests de actualizat (estimare: 1 zi)

Timeline: Sprint 24. Documentatie de migrare: [link]
Intrebari? Thread-ul asta sau in #angular-migration.
```

**Catre Product Managers:**
```
Migram la un sistem mai nou de server-side rendering in Angular.

Ce inseamna pentru utilizatori:
- Pagina se incarca vizibil la fel de repede
- DAR devine interactiva cu 40% mai repede (utilizatorul poate
  clicka butoane mai devreme)
- Elimina un bug vizual (flickering) pe care il avem acum

Effort: ~4 zile distribuite pe 3 echipe, in Sprint 24.
Nu afecteaza feature delivery — se face in paralel.
Risc: Minim, avem rollback plan.
```

**Catre Leadership (VP/CTO):**
```
Modernizam server-side rendering-ul pentru a imbunatati experienta
utilizatorilor si a reduce riscul tehnic.

Business impact:
- Conversie potentiala mai buna: pagini interactive cu 40% mai repede
  (studiile arata ca fiecare 100ms delay reduce conversia cu 1%)
- Reducerea riscului: migrare de pe o tehnologie deprecated
  care nu va mai primi security patches dupa Q4

Efort: 4 zile, 3 echipe. Zero impact pe delivery.
Masurare: Core Web Vitals (LCP, FID) inainte si dupa.
```

### Scrierea propunerilor tehnice clare

**Structura unui "one-pager" tehnic:**

```markdown
# Propunere: Implementarea Design System-ului cu Angular CDK

## TL;DR (max 3 randuri)
Construim un design system intern bazat pe Angular CDK care va
unifica UI-ul celor 5 aplicatii si va reduce timpul de dezvoltare
a componentelor noi cu ~50%.

## Problema (cu date)
- 5 aplicatii Angular, fiecare cu propriile componente UI
- 47 implementari de butoane, 23 de modals, 31 de forms
- Bug-urile UI se repara in fiecare aplicatie separat
- Timpul mediu de implementare a unei pagini noi: 5 zile
  (2 zile sunt reinventarea componentelor)

## Solutia propusa
[Diagrama]
[2-3 paragrafe concise]

## Ce NU include
[Explicit: ce e out of scope]

## Cost vs Benefit
| | Cost | Benefit |
|---|---|---|
| Timp | 6 saptamani, 2 devs | Reduce dev time cu 50% |
| Mentenanta | 1 dev part-time | Elimina duplicarea |
| Risc | Adoption lent | Consistency, quality |

## Next Steps
1. [Actiune] — [Owner] — [Deadline]
2. [Actiune] — [Owner] — [Deadline]
```

### Prezentarea trade-off-urilor obiectiv

**Framework pentru prezentarea trade-off-urilor:**

```
Pentru fiecare optiune, evalueaza pe aceste axe:

COMPLEXITY          [Low -------- High]
EFFORT              [Low -------- High]
RISK                [Low -------- High]
PERFORMANCE GAIN    [Low -------- High]
MAINTAINABILITY     [Low -------- High]
TEAM FAMILIARITY    [Low -------- High]

Exemplu: Alegerea strategiei de caching

Optiunea A: HTTP caching (ETag + Cache-Control)
  Complexity:       [==--------] Low
  Effort:           [==--------] Low
  Risk:             [==---------] Low
  Performance Gain: [====------] Medium
  Maintainability:  [========--] High
  Team Familiarity: [========--] High

Optiunea B: Service Worker caching (Angular PWA)
  Complexity:       [=====-----] Medium
  Effort:           [====------] Medium
  Risk:             [====------] Medium (cache invalidation)
  Performance Gain: [========--] High
  Maintainability:  [=====-----] Medium
  Team Familiarity: [===-------] Low

Optiunea C: Custom in-memory cache cu signals
  Complexity:       [=======---] High
  Effort:           [=======---] High
  Risk:             [======----] Medium-High
  Performance Gain: [==========-] Very High
  Maintainability:  [====------] Medium
  Team Familiarity: [==--------] Low

RECOMANDARE: Optiunea A ca baseline, cu Optiunea B adaugata
in faza 2 pentru offline support.
RATIONALE: Maxim value cu minim risk. Optiunea C ramane
optiune daca benchmark-urile arata ca A+B nu sunt suficiente.
```

### Folosirea diagramelor si vizualelor

**Reguli pentru diagrame tehnice eficiente:**

1. **Un concept per diagrama** — nu pune totul intr-o singura diagrama
2. **Labeluri clare** — fara ambiguitate
3. **Legenda** — daca folosesti culori sau forme diferite
4. **Flow direction** — de sus in jos sau de la stanga la dreapta
5. **Highlight-ul** — coloreaza diferit elementul central

**Tipuri de diagrame si cand le folosesti:**

| Tip | Cand | Tool |
|-----|------|------|
| Architecture diagram (C4) | Prezentari high-level | Mermaid, draw.io |
| Sequence diagram | API flows, interactiuni | Mermaid, PlantUML |
| Flowchart | Decision processes, logic | Mermaid |
| Entity-Relationship | Data models | dbdiagram.io |
| Dependency graph | Module relationships | Nx graph, Madge |

### Documentarea deciziilor si rationale-ului

**Nu documenta doar CE ai decis. Documenteaza DE CE.**

```
BAD:
"Folosim lazy loading pentru toate modulele."

GOOD:
"Folosim lazy loading pentru toate feature modules pentru ca:
1. Bundle-ul initial era 2.4MB (target: sub 500KB)
2. 70% din utilizatori acceseaza doar 2 din 8 feature-uri
3. Lazy loading reduce initial bundle la 380KB
4. Trade-off: prima navigare catre un feature e cu ~200ms mai lenta
   dar acceptabil daca adaugam preloading strategy"
```

### Weekly/Monthly tech updates

**Format de Tech Update saptamanal:**

```markdown
# Tech Update — Saptamana 7, 2026

## Highlight-ul saptamanii
Migrarea la Angular 19.1 s-a completat cu succes pe toate
cele 5 aplicatii. Zero regressions in productie.

## Metrici
- Build time mediu: 3.2 min (-15% vs saptamana trecuta)
- Test coverage: 78.3% (+1.2%)
- Bundle size (main app): 412KB (-8KB)
- Open PRs > 3 zile: 2 (target: 0)

## Decizii luate saptamana aceasta
- ADR-021: Adoptam Playwright pentru E2E (inlocuim Cypress)
- RFC-015: In review — Migrare la standalone components

## In progres
- [60%] Design system v2 (target: Sprint 26)
- [30%] Performance optimization dashboard (target: Sprint 27)

## Riscuri si blocaje
- [RISK] Angular 20 va depreca ngModules — trebuie accelerata
  migrarea la standalone (RFC-015)
- [BLOCKED] CI pipeline instabil — investigare in curs

## Urmatoarea saptamana
- Finalizarea RFC-015 review
- Inceperea PoC pentru micro-frontends
```

---

## 5. Gestionarea conflictelor tehnice

### Disagree and Commit

**Principiu fundamental:** Dupa o discutie onesta, echipa ia o decizie. Chiar daca nu esti de acord, te angajezi 100% la implementarea deciziei. Nu sabotezi, nu spui "v-am zis eu".

```
EXEMPLU DE APLICARE CORECTA:

Context: Echipa trebuie sa aleaga intre REST si GraphQL.
Tu preferi GraphQL, dar echipa alege REST.

GRESIT:
"Ok, faceti cum vreti, dar cand o sa aveti probleme cu
over-fetching, sa nu veniti la mine."

CORECT:
"Am prezentat argumentele pentru GraphQL si am ascultat
contra-argumentele. Echipa a decis REST pe baza experientei
existente si a timeline-ului strans. Accept decizia si voi
ajuta sa implementam REST-ul cat mai bine. Am documentat
alternativa in ADR-ul nostru pentru referinta viitoare."

DE CE FUNCTIONEAZA:
- Respecti inteligenta echipei
- Pastrezi coeziunea grupului
- Documentezi alternativa (poate reveni in viitor)
- Demonstrezi maturitate si leadership
```

**Cand NU aplici "disagree and commit":**
- Decizia prezinta **riscuri de securitate** reale
- Decizia **incalca reglementari** (GDPR, compliance)
- Decizia va cauza **pierderi de date** sau **downtime major**
- In aceste cazuri: **escaleaza**, nu te conforma

### Data-driven Decision Making

**Cum transformi o dezbatere subiectiva in una obiectiva:**

```
SCENARIU: "Trebuie sa rescriem aplicatia din scratch" vs
"Trebuie sa refactorizam incremental"

INAINTE (opinie vs opinie):
Dev A: "Codul e un dezastru, trebuie rescris."
Dev B: "Rescrierea e prea riscanta, trebuie refactorizat."
(Se repeta la infinit, nimeni nu convince pe nimeni)

DUPA (date vs date):
Principal Engineer: "Hai sa masuram inainte sa decidem."

METRICI COLECTATE:
- Timpul mediu de adaugare a unui feature nou: 2 saptamani
  (target acceptabil: 1 saptamana)
- Numarul de bug-uri per feature: 3.2 (target: <1)
- Acoperirea cu teste: 12% (target: >70%)
- Timpul de build: 12 minute (target: <3 min)
- Developer satisfaction (survey): 2.1/5

ANALIZA:
- Rescrierea: 6 luni, 4 devs, risc de re-creare a bug-urilor,
  zero features noi in aceasta perioada
- Refactoring incremental: 12 luni pana la target, dar livrezi
  features in paralel, risc mai mic

DECIZIA (bazata pe date):
"Refactoring incremental cu 'strangler fig pattern' — migrarea
modulara, feature cu feature, cu teste adaugate inainte de
fiecare migrare. In 3 luni facem un checkpoint — daca progresul
e sub 30%, reconsideram rescrierea."
```

### Proof of Concept (PoC) pentru rezolvarea dezbaterilor

**Cand sa faci un PoC:**
- Doua abordari par viabile si nimeni nu poate argumenta definitiv
- Diferenta de performanta e cruciala dar necunoscuta
- Echipa nu are experienta cu tehnologia propusa
- Riscul e prea mare pentru a decide doar pe baza teoriei

**Structura unui PoC eficient:**

```
REGULI:
1. Time-boxed: Maximum 2-3 zile
2. Scope limitat: Rezolva O SINGURA intrebare
3. Criteriu de succes: Definit INAINTE de a incepe
4. Prezentare: Rezultate partajate cu toti stakeholders

EXEMPLU: "SSR cu Angular Universal vs SSR cu Analog.js"

Intrebarea: Care solutie ofera LCP mai bun pe pagina de produs?

PoC Plan:
- Ziua 1: Setup ambele variante cu pagina de produs reala
- Ziua 2: Benchmark cu Lighthouse si WebPageTest (5 runs fiecare)
- Ziua 3: Documentare rezultate si prezentare

Criteriu de succes:
- LCP sub 1.5s pe 3G simulat
- Hidration completa sub 2s
- DX acceptabil (build time, debugging experience)

Rezultate:
| Metric | Universal | Analog.js |
|--------|-----------|-----------|
| LCP (3G) | 1.8s | 1.3s |
| Hydration | 1.5s | 0.9s |
| Build time | 45s | 62s |
| DX | Familiar | Nou, necesita invatare |

Decizia: Analog.js castiga pe performance dar echipa nu il
cunoaste. Recomandam Universal acum cu plan de migrare la
Analog.js in Q3, cand Angular 20 il va face mai matur.
```

### Facilitarea discutiilor tehnice productive

**Framework "RAPID" pentru discutii tehnice:**

```
R - RESTATE the problem (2 min)
    "Suntem aici sa decidem X. Problema e Y. Constrangerile sunt Z."

A - ALTERNATIVES listing (10 min)
    Fiecare persoana propune solutii. Nu se critica inca.
    Scrie pe whiteboard/doc.

P - PROS/CONS analysis (15 min)
    Pentru fiecare alternativa, echipa identifica pro/con.
    Facilitatorul se asigura ca fiecare alternativa e evaluata
    corect (nu doar favoritele).

I - IDENTIFY the decision (5 min)
    "Pe baza analizei, eu recomand X pentru ca Y."
    Sau: "Avem nevoie de PoC/date aditionale inainte de decizie."

D - DOCUMENT and DECIDE (5 min)
    Scrie decizia in ADR.
    Stabileste owner si next steps.
    "Exista cineva care nu poate face 'disagree and commit'?"
```

**Reguli de facilitare:**
- **O voce la un moment dat** — foloseste un "token" (obiect fizic sau virtual)
- **Timeboxing** — fiecare sectiune are un timer
- **"Strongest argument against"** — cere-i fiecaruia sa prezinte cel mai bun argument IMPOTRIVA propriei pozitii
- **Silent writing** — pentru a evita groupthink, fiecare scrie pozitia initial in tacere
- **Votare anonima** — pentru decizii sensibile

### Cand sa escalezi vs cand sa rezolvi local

```
REZOLVA LOCAL cand:
- Impactul e limitat la echipa ta
- Decizia e reversibila
- Ai datele necesare pentru a decide
- Stakeholders sunt de acord cu directia
- Nu exista implicatii de securitate sau compliance

ESCALEAZA cand:
- Impactul e cross-team sau cross-departament
- Decizia e ireversibila sau foarte costisitoare
- Exista un conflict persistent care blocheaza echipa
- Sunt implicatii legale, de securitate sau compliance
- Nu ai autoritatea de a aloca resursele necesare
- Doua echipe au nevoi conflictuale

CUM ESCALEZI CORECT:
1. Prezinti problema clar, cu context
2. Prezinti optiunile evaluate (cu pro/con)
3. Faci o recomandare clara
4. Specifici ce ai nevoie: decizie? resurse? aliniere?
5. NU escalezi fara sa fi incercat sa rezolvi local mai intai

EXEMPLU:
"Am o situatie care necesita input-ul tau. Echipa Payments
si echipa Checkout au nevoie de abordari diferite pentru
authentication flow. Am explorat 3 optiuni [link ADR].
Recomandarea mea e Optiunea B dar echipa Payments nu e de
acord din cauza [motiv valid]. Am nevoie de ajutorul tau
pentru a facilita o decizie finala."
```

### Managing Egos in Technical Debates

**Strategii practice:**

```
1. SEPARATE IDEEA DE PERSOANA
   Gresit: "Abordarea lui Alex e gresita."
   Corect: "Abordarea A are trade-off-ul X. Abordarea B are
   trade-off-ul Y. Cum decidem?"

2. FOLOSESTE "AND", NU "BUT"
   Gresit: "Ideea ta e interesanta, DAR nu va functiona."
   Corect: "Ideea ta rezolva problema X, SI trebuie sa gandim
   cum adresam si problema Y."

3. RECUNOASTE CONTRIBUTIA
   "Punctul lui Andrei despre caching e foarte bun si trebuie
   incorporat indiferent de solutia aleasa."

4. REDIRECTIONEAZA CATRE DATE
   "Amandoi avem argumente puternice. Hai sa definim un
   experiment care sa ne dea un raspuns obiectiv."

5. OFERA 'SALVARE DE IMAGINE'
   "Cu informatiile pe care le aveam luna trecuta, decizia
   originala era corecta. Acum avem date noi care schimba
   calculul."

6. PRIVATE CONVERSATIONS
   Daca cineva blocheaza o decizie din ego, vorbeste 1-on-1.
   "Am observat ca ai o opinie puternica despre X. Ajuta-ma sa
   inteleg preocuparea ta principala — poate gasim o cale."

7. "STRONG OPINIONS, LOOSELY HELD"
   Modeleaza tu insuti: "Am crezut ca Y e cel mai bun,
   dar argumentul lui Z m-a convins sa reconsider."
```

---

## 6. Influenta fara autoritate directa

### Building Trust Through Competence

Influenta unui Principal Engineer vine din **credibilitate**, nu din titlu.

```
PILONII CREDIBILITATII TEHNICE:

1. EXPERTISE DEMONSTRATA
   - Rezolvi probleme pe care altii nu le pot rezolva
   - Raspunsurile tale la intrebari tehnice sunt corecte si nuantate
   - Code review-urile tale adauga valoare reala
   - Exemplu: "Cand echipa X a avut o problema de memory leak,
     am identificat cauza in 2 ore si am propus o solutie
     generalizabila pentru toata organizatia"

2. RELIABILITY
   - Faci ce spui ca faci
   - Respecti deadline-urile pe care le stabilesti
   - Cand nu poti, comunici proactiv
   - Exemplu: "Am promis un RFC pana vineri si l-am livrat.
     De fiecare data."

3. TRANSPARENCY
   - Recunosti cand nu stii ceva
   - Esti deschis despre trade-off-uri
   - Nu ascunzi problemele
   - Exemplu: "Nu am experienta cu Kubernetes, dar pot
     invata. Intre timp, echipa Y are expertise — sugereaza
     sa colaboram."

4. CONSISTENCY
   - Aceleasi standarde pentru toata lumea (inclusiv tine)
   - Nu schimbi pozitia in functie de cine asculta
   - Feedback consistent si predictibil
```

### Leading by Example

```
ACTIUNI CARE DEMONSTREAZA LEADERSHIP:

Cod:
- Scrii teste pentru fiecare feature pe care il faci
- Faci refactoring-uri cand vezi cod problematic
- Folosesti pattern-urile pe care le recomanzi altora
- Contribui la tooling-ul intern

Documentatie:
- ADR-urile tale sunt modele pe care altii le copiaza
- Scrii post-mortem-uri detaliate si utile
- README-urile proiectelor tale sunt complete
- Comentariile in cod explica "de ce", nu "ce"

Procese:
- Raspunzi la code reviews in <24 ore
- Participi activ la on-call rotation
- Faci pair programming cu developeri din alte echipe
- Iti actualizezi documentation cand schimbi codul

IMPACT:
Cand altii vad ca tu (Principal Engineer) faci aceste lucruri,
le adopta natural. "Daca el/ea face asta, inseamna ca e
important."
```

### Crearea coalitiilor de suport

**Cum convingi o organizatie sa adopte o schimbare:**

```
STRATEGIE "SNOWBALL":

1. IDENTIFICA EARLY ADOPTERS (Saptamana 1-2)
   - Gaseste 2-3 engineers care au aceeasi problema
   - Propune solutia informal
   - Obtine feedback si ajusteaza

2. CREAZA UN PILOT (Saptamana 3-4)
   - Implementeaza solutia cu un early adopter
   - Colecteaza metrici si feedback
   - Documenta rezultatele

3. DEMONSTRATE VALUE (Saptamana 5-6)
   - Prezinta rezultatele pilot-ului
   - Lasa early adopters sa povesteasca experienta lor
   - "Nu eu spun ca e bun — colegii vostri spun"

4. EXTEND GRADUALLY (Saptamana 7+)
   - Invita urmatoarea echipa
   - Ofera suport activ in adoptare
   - Colecteaza feedback si itereaza
   - Repeta pana atinge masa critica

EXEMPLU CONCRET:
"Am vrut sa introduc Nx monorepo. Nu am propus un RFC imediat.
Am migrat intai proiectul meu ca pilot. Dupa 2 saptamani,
build time-ul a scazut de la 8 min la 2 min. Am aratat asta
echipei vecine, care a vrut si ea. Dupa 1 luna, 3 echipe
foloseau Nx. Abia atunci am scris RFC-ul — cu date de la 3
echipe, aprobarea a fost imediata."
```

### Incremental Influence (start small, prove value)

```
PRINCIPIU: Nu propune revolutii. Propune experimente.

GRESIT:
"Trebuie sa migram totul la signals. Iata RFC-ul de 20 pagini."
(Reactia: rezistenta, frica, "merge si asa cum e")

CORECT:
Pasul 1: "Pot sa incerc signals pe o componenta? 2 ore de lucru."
Pasul 2: "A functionat, iata rezultatele. Mai incerc pe 3?"
Pasul 3: "5 componente migrate, performanta e cu 20% mai buna."
Pasul 4: "Cine vrea sa incerce? Pot ajuta cu pair programming."
Pasul 5: "10 componente migrate de 3 echipe. Scriem un standard?"
(Reactia: "Da, evident, deja functoineaza")

DE CE FUNCTIONEAZA:
- Oamenii accepta mai usor "hai sa incercam" decat "trebuie sa schimbam"
- Datele reale conving mai bine decat argumentele teoretice
- Oamenii vor sa fie parte din ceva care deja functioneaza
- Nici un manager nu blocheaza un experiment de 2 ore
```

### Technical Blog Posts si Internal Talks

**Formate de thought leadership:**

```
1. TECH BLOG INTERN (Confluence, Notion, Wiki)
   - Frecventa: 1-2 articole pe luna
   - Topicuri: Lessons learned, comparatii, how-to-uri
   - Exemplu: "De ce am ales Signal Store peste NgRx Store clasic"
   - Impact: Creeaza referinta permanenta, demonstreaza gandire

2. LIGHTNING TALKS (15 min)
   - Frecventa: Saptamanal, rotatie
   - Format: O idee, demonstrata practic
   - Exemplu: "Angular inject() function — 5 patterns pe care
     nu le cunosteai"
   - Impact: Invatare continua, vizibilitate

3. ARCHITECTURE DEEP DIVES (60 min)
   - Frecventa: Lunar
   - Format: Prezentare detaliata + Q&A
   - Exemplu: "Cum am redus bundle size-ul cu 60%"
   - Impact: Transfer de cunostinte profund

4. LUNCH AND LEARN
   - Frecventa: Bi-saptamanal
   - Format: Informal, discutie libera pe un topic
   - Exemplu: "Ce parere avem despre micro-frontends?"
   - Impact: Construieste comunitate, identifica interese

5. EXTERNAL TALKS (Conferinte, Meetups)
   - Frecventa: 2-4 pe an
   - Impact: Brand personal + branding companie
   - Beneficiu intern: "Colegul nostru e speaker la ngConf"
```

### Being the Go-to Person

```
CUM DEVII PERSOANA LA CARE APELEAZA TOATA LUMEA:

1. RASPUNDE RAPID
   - Mesajele pe Slack: <2 ore
   - Code review-urile: <24 ore
   - Intrebarile directe: imediat sau "revin in 1 ora"

2. RASPUNDE UTIL
   - Nu doar raspunsul, ci si context-ul
   - Nu doar solutia, ci si de ce e cea mai buna
   - Include link-uri, documentatie, exemple

3. RASPUNDE GENEROS
   - Nu "RTFM", ci "iata link-ul si punctul cheie"
   - Nu "e simplu", ci explicatie la nivelul interlocutorului
   - Share-uieste cunostintele fara sa tii cont

4. CONSTRUIESTE KNOWLEDGE BASE
   - Raspunsurile frecvente le transformi in documentatie
   - FAQ intern actualizat constant
   - Runbook-uri pentru scenarii comune

5. REDIRECTIONEAZA CORECT
   - Daca nu stii: "Nu stiu, dar X e expert pe asta"
   - Nu pretinde ca stii totul
   - Conecteaza oamenii intre ei
```

---

## 7. Code Reviews eficiente

### Ce sa cauti in code review (prioritizat)

```
PRIORITATEA 1 — SHOW STOPPERS (blocheaza aprobarea)
================================================
[ ] Corectitudine: Codul face ce trebuie sa faca?
[ ] Securitate: XSS, injection, auth bypass, data exposure?
[ ] Data loss: Se pot pierde date in orice scenariu?
[ ] Performance critice: N+1 queries, memory leaks, infinite loops?

PRIORITATEA 2 — PROBLEME IMPORTANTE
===================================
[ ] Arhitectura: Respecta pattern-urile echipei? Single responsibility?
[ ] Error handling: Sunt toate erorile tratate? Ce se intampla cand fail-uieste?
[ ] Edge cases: Null, empty, concurrency, timeout?
[ ] Teste: Exista? Acopera cazurile relevante? Sunt corecte?
[ ] API design: Interfetele sunt intuitive si consistente?

PRIORITATEA 3 — IMBUNATATIRI
=============================
[ ] Naming: Variabile, functii, fisiere au nume clare?
[ ] Complexitate: Se poate simplifica? Functii prea lungi?
[ ] DRY: Exista duplicare care ar trebui extrasa?
[ ] Documentatie: Codul complex are comentarii "de ce"?
[ ] Type safety: TypeScript types sunt corecte si specifice?

PRIORITATEA 4 — NIT (optional, nu blocheaza)
=============================================
[ ] Consistenta stilistica (daca nu e acoperita de linter)
[ ] Formatare minora
[ ] Ordering imports (daca nu e automatizat)
```

### Ce sa NU cauti in code review

```
NU PIERDE TIMP PE:
- Formatting (ESLint + Prettier rezolva asta automat)
- Import ordering (eslint-plugin-import)
- Trailing spaces, semicolons, quotes
- Naming conventions mecanic verificabile (lint rules)
- Orice poate fi automatizat — AUTOMATIZEAZA-L

ANTI-PATTERN-URI DE CODE REVIEW:
- "Eu as fi facut diferit" (fara motiv concret)
- Rewrite-uri complete in review (propune, nu impune)
- Bike-shedding pe detalii minore
- Review-uri care dureaza mai mult decat codul in sine
- Tone policing (concentreaza-te pe cod, nu pe stilul PR description)
```

### Oferirea feedback-ului constructiv

```
FORMULA: Observatie + Impact + Sugestie

GRESIT:
"Acest cod e prost."
"De ce nu ai folosit signals?"
"Asta nu e cum se face."

CORECT:
"[Observatie] Vad ca folosesti BehaviorSubject manual aici.
[Impact] Asta inseamna ca trebuie sa gestionam manual subscription
lifecycle si nu beneficiem de OnPush change detection.
[Sugestie] Ce zici de un computed signal? Ar simplifica codul si
ar fi mai performant. Exemplu: `count = computed(() => this.items().length)`"

NIVELURI DE FEEDBACK:
- "nit:" — sugestie minora, nu blocheaza aprobarea
- "suggestion:" — recomandare, ar fi bine dar nu obligatoriu
- "issue:" — problema reala, trebuie adresata
- "blocker:" — nu pot aproba fara fix
- "question:" — nu inteleg, clarifica te rog
- "praise:" — "Excelent! Pattern-ul asta rezolva elegant problema X"

EXEMPLU COMPLET:
```

```
// suggestion: Acest subscription ar putea fi un memory leak.
// In componenta Angular, subscription-urile trebuie cleanup.
// Sugestie: foloseste takeUntilDestroyed() sau toSignal():
//
// Inainte:
this.service.getData().subscribe(data => this.data = data);

// Dupa (varianta signals):
this.data = toSignal(this.service.getData(), { initialValue: [] });

// Dupa (varianta clasica):
this.service.getData()
  .pipe(takeUntilDestroyed())
  .subscribe(data => this.data = data);
```

### PR Size Guidelines

```
RECOMANDARI:

Dimensiune ideala:  50-200 linii modificate
Maximum acceptabil: 400 linii
Peste 400 linii:    Cere split-ul PR-ului

DE CE:
- PR-uri mici = review mai atent = mai putine bug-uri
- PR-uri mari = review superficial ("looks good") = bug-uri
- Studiu (Microsoft): Review quality scade dramatic dupa 200 linii
- Studiu (SmartBear): La 400+ linii, rata de detectie a bug-urilor
  scade la <20%

CUM SA SPARGI PR-URI MARI:
1. Feature flag-uri: codul e in productie dar inactiv
2. Vertical slicing: o functionalitate completa (UI + logic + test)
   mai degraba decat "toate serviciile" apoi "toate componentele"
3. Preparatory refactoring: un PR de refactoring, apoi un PR de feature
4. Stacked PRs: PR-uri mici care depind unul de altul

EXCEPTII ACCEPTABILE PENTRU PR-URI MARI:
- Migrari automate (ex: ng update)
- Generare de cod (ex: Nx generators)
- Redenumiri la scara larga (cu tool automat)
- Adaugarea de teste pe cod existent
```

### Review Turnaround Time

```
ASTEPTARI:

PR mic (<100 linii):     Review in <4 ore
PR mediu (100-300 linii): Review in <8 ore (aceeasi zi)
PR mare (300+ linii):     Review in <24 ore
PR urgent (hotfix):       Review in <1 ora

DE CE CONTEZA:
- PR-uri care stau in review = context switching costisitor
- Developer-ul trece la alt task, apoi trebuie sa revina
- Studiu: fiecare zi de asteptare in review adauga ~1 zi la
  delivery total (din cauza context switching)

STRATEGII PENTRU TURNAROUND RAPID:
1. Blocheaza 2x30 min pe zi doar pentru reviews (dimineata + dupa-amiaza)
2. Review the oldest PR first (FIFO)
3. Daca nu poti face review complet, ofera feedback partial:
   "Am vazut prima parte, totul ok. Revin maine cu restul."
4. Seteaza notificari pentru PR assignments
5. Rotatie de "review duty" in echipa
```

### Teaching Through Code Review

```
CODE REVIEW CA INSTRUMENT DE INVATARE:

1. EXPLICA "DE CE", NU DOAR "CE"
   Gresit: "Foloseste trackBy aici."
   Corect: "Fara trackBy, Angular va re-renda toata lista la
   fiecare change detection cycle, chiar daca doar un item s-a
   schimbat. Cu trackBy, Angular identifica elementele noi/sterse
   si face DOM updates minime. Pe o lista de 1000 items,
   diferenta e de la 200ms la 5ms."

2. OFERA RESURSE
   "Pattern-ul asta se cheama 'facade' si e foarte util in Angular
   pentru a abstractiza state management. Articol detaliat: [link]"

3. SHARE ALTERNATIVE
   "Codul tau functioneaza corect. Alt mod de a face asta ar fi
   cu computed signals — iata cum ar arata: [exemplu].
   Nu e necesar sa schimbi acum, dar e bine de stiut."

4. RECUNOASTE PROGRESUL
   "Vad ca ai aplicat pattern-ul de care am discutat saptamana
   trecuta. Excelent!"

5. ADRESEAZA INTREBARI SOCRATICE
   "Ce s-ar intampla daca acest Observable nu emite niciodata?
   Cum ar afecta componenta?"
```

### Code Review Checklist

```markdown
## Code Review Checklist — Angular

### Basics
- [ ] PR description explica CE si DE CE
- [ ] PR-ul e legat de ticket/issue
- [ ] Dimensiunea e rezonabila (<400 linii)
- [ ] Branch-ul e up-to-date cu main

### Corectitudine
- [ ] Codul face ce spune PR description ca face
- [ ] Edge cases sunt tratate (null, empty, error)
- [ ] Nu exista regressions pe functionalitatea existenta

### Angular-specific
- [ ] Components folosesc OnPush change detection
- [ ] Subscriptions sunt cleanup-uite (takeUntilDestroyed, toSignal, async pipe)
- [ ] Lazy loading unde e posibil
- [ ] trackBy pe *ngFor / @for
- [ ] Signals folosite corect (computed pentru derived state)
- [ ] Injectia de dependinte e corecta (providedIn, provide scope)

### Security
- [ ] Input sanitization (DomSanitizer unde e necesar)
- [ ] Nu exista XSS (innerHTML doar cu sanitize)
- [ ] Auth guards pe rute protejate
- [ ] Sensitive data nu e logata sau expusa

### Performance
- [ ] Nu sunt apeluri in template care recalculeaza la fiecare CD
- [ ] Bundle impact considerat (import-uri tree-shakeable)
- [ ] Nu sunt memory leaks (subscriptions, event listeners, intervals)
- [ ] Imagini au loading="lazy" unde e potrivit

### Testing
- [ ] Unit tests pentru logica noua
- [ ] Tests verifica behavior, nu implementare
- [ ] Edge cases testate
- [ ] Mocking-ul e minimal si corect

### Maintainability
- [ ] Naming clar si consistent
- [ ] Functii scurte (<30 linii ideal)
- [ ] Nu exista cod duplicat
- [ ] TypeScript types sunt specifice (nu 'any')
- [ ] Comentarii pe "de ce" (nu pe "ce")
```

---

## 8. Driving Technical Vision

### Technology Radar

**Technology Radar** este un instrument de vizualizare a tehnologiilor relevante pentru organizatie, categorisite in 4 zone:

```
STRUCTURA:

ADOPT (folosim activ si recomandam)
  - Angular 19+
  - NgRx Signal Store
  - Nx Monorepo
  - Playwright (E2E)
  - TypeScript strict mode

TRIAL (evaluam activ, proiecte pilot)
  - Angular Signals (migrare de la RxJS)
  - Analog.js (meta-framework)
  - Module Federation (micro-frontends)
  - Bun (build tool)

ASSESS (monitorizam, merita atentie)
  - Angular Wiz (hybrid rendering)
  - TC39 Decorators (stage 3)
  - Import maps (native ES modules)
  - Rspack (Webpack replacement)

HOLD (nu adoptam, sau retragem)
  - AngularJS (legacy, migrare in curs)
  - Protractor (inlocuit cu Playwright)
  - Karma (inlocuit cu Jest/Vitest)
  - Zone.js (se retrage in Angular 20+)

ACTUALIZARE: Trimestrial
OWNER: Principal Engineer + Tech Leads
PROCESS: Oricine propune schimbari, discutie in guild meeting
```

### Definirea standardelor si best practices

**Cum creezi standarde care se adopta (nu se ignora):**

```
1. COLLABORATIVELY, NOT TOP-DOWN
   - Nu dicteaza: "De acum folosim X"
   - Faciliteaza: "Ce pattern folosim pentru Y? Hai sa standardizam"
   - Implica echipele in crearea standardelor

2. DOCUMENT WITH EXAMPLES
   Nu doar: "Folositi facade pattern"
   Ci:

   ```typescript
   // STANDARD: Fiecare feature module expune un facade service
   // care abstractizeaza state management

   @Injectable({ providedIn: 'root' })
   export class ProductsFacade {
     // Public signals (read-only)
     readonly products = this.store.products;
     readonly loading = this.store.loading;
     readonly error = this.store.error;

     // Computed
     readonly activeProducts = computed(() =>
       this.products().filter(p => p.isActive)
     );

     constructor(private store: ProductsStore) {}

     // Public methods (actions)
     loadProducts(): void { this.store.loadAll(); }
     addProduct(product: CreateProductDto): void {
       this.store.add(product);
     }
   }
   // Componenta NICIODATA nu acceseaza store-ul direct.
   // Componenta NICIODATA nu apeleaza HTTP direct.
   ```

3. AUTOMATE ENFORCEMENT
   - ESLint rules custom pentru standardele voastre
   - Schematics/generators Nx pentru boilerplate
   - PR templates cu checklist
   - CI checks automate

4. GOLDEN PATH TEMPLATES
   - "Cand creezi un feature nou, foloseste: nx g @org/plugin:feature"
   - Template-ul include: structura, teste, facade, store
   - Developer-ul incepe de la 80% complet, nu de la 0%
```

### Leading Migration and Modernization Efforts

**Framework pentru migrari la scara larga:**

```
FAZA 1: ASSESSMENT (1-2 saptamani)
===================================
- Inventar: Ce avem acum? (versiuni, patterns, dependinte)
- Impact: Ce trebuie schimbat? (numar fisiere, echipe afectate)
- Risc: Ce poate merge gresit? (breaking changes, regressions)
- Effort: Cat dureaza? (estimare pe faze)

FAZA 2: STRATEGY (1 saptamana)
===============================
- Abordare: Big bang vs incremental vs strangler fig?
- Prioritizare: Ce se migreaza prima? (cel mai folosit? cel mai risky?)
- Tooling: Ce tools ne ajuta? (ng update, schematics, codemods)
- Testing: Cum verificam ca n-am stricat nimic?

FAZA 3: PILOT (2-4 saptamani)
==============================
- Migreaza 1 modul/aplicatie ca proof of concept
- Documenteaza problemele intalnite
- Creeaza runbook pentru migrare
- Valideaza estimarile

FAZA 4: ROLLOUT (variabil)
===========================
- Migreaza in waves (echipa cu echipa)
- Suport activ pentru fiecare echipa
- Daily standup de migrare (15 min)
- Dashboard cu progresul migrarii

FAZA 5: CLEANUP (1-2 saptamani)
================================
- Sterge codul vechi
- Actualizeaza documentatia
- Retro: ce am invatat?
- Actualizeaza Technology Radar

EXEMPLU: Migrare de la NgModules la Standalone Components
=========================================================
Assessment: 847 componente, 52 module, 12 echipe
Strategy: Incremental, bottom-up (leaf components first)
Tooling: ng g @angular/core:standalone (schematic oficial)
Pilot: Feature-ul "User Profile" (23 componente)
Rollout: 4 waves de cate 3 echipe
Timeline: 8 saptamani
```

### Evaluarea tehnologiilor noi

**Framework "TRICE" pentru evaluare:**

```
T — TECHNICAL FIT
  - Rezolva problema noastra specifica?
  - Se integreaza cu stack-ul actual?
  - Performance benchmarks relevante?

R — RISK
  - Cat de matur e proiectul? (Stars, contributors, releases)
  - Cine il mentine? (companie vs community)
  - Breaking changes frecvente?
  - Lock-in risk?

I — INVESTMENT
  - Curba de invatare pentru echipa?
  - Efort de migrare?
  - Cost (licente, infrastructure)?
  - Mentenanta pe termen lung?

C — COMMUNITY
  - Ecosistem de librarii/plugins?
  - Documentatie de calitate?
  - StackOverflow / GitHub Issues activity?
  - Conferinte, cursuri, carti?

E — EVOLUTION
  - Roadmap clar si sustinut?
  - Aliniat cu directia industriei?
  - Backward compatibility policy?
  - Alternativa daca proiectul moare?

SCOR: Fiecare criteriu 1-5, total minim 15/25 pentru adoptare.
```

### Building a Technical Roadmap

```
EXEMPLU: Technical Roadmap Angular — 2026

Q1 2026 (FOUNDATION)
=====================
[x] Migrare Angular 19 (toate aplicatiile)
[x] Adoptare Signal Store
[ ] Standardizare testing (Jest -> Vitest)
[ ] Design system v2 launch
    KR: Bundle size < 500KB per app
    KR: Build time < 3 min per app

Q2 2026 (MODERNIZATION)
========================
[ ] Eliminare Zone.js (signals everywhere)
[ ] Standalone components 100%
[ ] Micro-frontend pilot (Module Federation)
[ ] Performance monitoring (Core Web Vitals dashboard)
    KR: LCP < 1.5s pe toate paginile
    KR: Zero NgModules in codebase

Q3 2026 (SCALE)
================
[ ] Micro-frontend rollout (3 echipe)
[ ] SSR cu hydration pe pagini critice
[ ] CI/CD optimization (< 10 min total pipeline)
[ ] Developer experience survey score > 4/5
    KR: Deployment frequency: daily
    KR: Lead time < 2 ore

Q4 2026 (INNOVATION)
=====================
[ ] Angular 20 adoption
[ ] AI-assisted development tooling
[ ] Edge rendering pilot
[ ] Mobile web optimization (PWA)
    KR: Lighthouse score > 95 pe toate paginile
    KR: Zero tech debt items marked "critical"

NOTITE:
- Roadmap-ul se revizuieste trimestrial
- KR = Key Results (masurabile)
- Prioritatea se ajusteaza pe baza feedback-ului business
```

### Aligning Technical Vision with Business Goals

```
FRAMEWORK: "BUSINESS GOAL -> TECHNICAL STRATEGY"

Exemplu 1:
Business: "Vrem sa cream pe piata din Germania"
Technical: -> i18n/l10n infrastructure
           -> CDN optimization pentru Europa
           -> GDPR compliance tehnic
           -> Locale-aware testing

Exemplu 2:
Business: "Vrem sa reducem churn-ul cu 15%"
Technical: -> Performance optimization (pagini rapide = useri fericiti)
           -> Error monitoring avansat (detectam problemele inainte de useri)
           -> A/B testing infrastructure
           -> Progressive enhancement (functioneaza si pe conexiuni lente)

Exemplu 3:
Business: "Vrem sa reducem costurile de development cu 20%"
Technical: -> Design system (reduce duplicarea)
           -> Nx monorepo (share code intre echipe)
           -> CI/CD optimization (developerii asteapta mai putin)
           -> Developer experience improvements (tooling, DX)

CUM COMUNICI ASTA:
Nu spui: "Trebuie sa migram la signals"
Spui: "Migrarea la signals va reduce timpul de development al
feature-urilor noi cu 20%, ceea ce inseamna ca putem livra
obiectivul de Q3 cu 2 saptamani mai devreme."

METRICI TEHNICE CARE INTERESEAZA BUSINESS-UL:
- Deployment frequency (cat de des putem livra)
- Lead time (cat de repede de la idee la productie)
- Mean time to recovery (cat de repede rezolvam probleme)
- Change failure rate (cate deployments cauzeaza probleme)
(Acestea sunt DORA metrics)
```

---

## 9. Intrebari comportamentale frecvente (STAR method)

### Ce este metoda STAR

```
S — SITUATION: Contextul si background-ul
    "In echipa de 8 developeri la compania X..."
    Clar, concis, relevant.

T — TASK: Responsabilitatea ta specifica
    "Eu eram responsabil pentru..."
    CE trebuia facut si DE CE era important.

A — ACTION: Ce ai facut concret TU
    "Am decis sa... Am implementat... Am facilitat..."
    Actiuni specifice, nu generalitati. Foloseste "EU", nu "NOI".

R — RESULT: Rezultatul masurabil
    "Ca rezultat, am redus X cu Y%..."
    Metrici concrete. Ce ai invatat. Ce ai face diferit.

DURATA RASPUNSULUI: 2-3 minute.
Nu mai scurt (pare superficial), nu mai lung (pierde atentia).
```

### 10 Intrebari comportamentale cu framework STAR

---

#### 1. "Tell me about a time you disagreed with a technical decision"

```
SITUATION:
Echipa de platform a decis sa foloseasca GraphQL pentru toate
API-urile interne, inclusiv pentru comunicarea service-to-service.

TASK:
Ca Principal Engineer, trebuia sa evaluez impactul pe toate
echipele frontend si sa ma asigur ca decizia e optima.

ACTION:
- Am analizat use case-urile: 70% erau CRUD simple, doar 30%
  beneficiau de flexibilitatea GraphQL
- Am pregatit un document cu date: overhead-ul GraphQL pe
  apeluri simple (latenta +15ms, complexitate schema)
- Am propus o abordare hibrida: GraphQL pentru BFF (Backend
  for Frontend), REST pentru service-to-service
- Am organizat un workshop cu ambele echipe, am prezentat
  datele obiectiv
- Am ascultat contra-argumentele si am ajustat propunerea

RESULT:
- Echipa a adoptat abordarea hibrida
- Latenta medie a scazut cu 12% fata de full-GraphQL
- Developer experience s-a imbunatatit (fiecare echipa
  foloseste tool-ul potrivit)
- Am documentat decizia intr-un ADR referentiat de 4 echipe

INVATAMINTE:
"Am invatat ca dezacordurile tehnice se rezolva cel mai bine
cu date, nu cu opinii. Si ca solutia optima e adesea un
compromis intre extremele propuse initial."
```

---

#### 2. "Describe a project that failed and what you learned"

```
SITUATION:
Am condus migrarea unei aplicatii Angular de la monolith la
micro-frontends. Proiectul a esuat in sensul ca am depasit
timeline-ul cu 3 luni si am introdus regressions.

TASK:
Eram owner-ul tehnic al migrarii, responsabil de arhitectura
si coordonarea celor 4 echipe implicate.

ACTION:
- Am identificat 3 cauze root ale esecului:
  1. Nu am facut un pilot suficient de mare (pilot-ul era o
     pagina simpla, productia avea pagini complexe)
  2. Am subestimat shared state intre micro-frontends
  3. Nu am definit un contract clar intre echipe
- Am oprit migrarea la jumatate si am propus un plan revizuit
- Am organizat un post-mortem cu toate echipele
- Am documentat lectiile invatate intr-un ADR si un blog post intern

RESULT:
- Migrarea s-a completat cu 3 luni intarziere dar fara regressions
  dupa planul revizuit
- Lectiile invatate au fost aplicate la urmatoarea migrare
  (standalone components) care s-a completat la timp
- Am creat un "Migration Playbook" folosit de toata organizatia

INVATAMINTE:
"Esecul m-a invatat ca migrarea in sine e partea usoara —
partea grea e intelegerea interdependintelor. De atunci,
inainte de orice migrare, fac un 'dependency mapping' complet
si pilot-ul trebuie sa acopere cel mai complex scenariu,
nu cel mai simplu."
```

---

#### 3. "How did you handle a situation where your team was resistant to change"

```
SITUATION:
Am vrut sa introduc TypeScript strict mode in toata organizatia.
3 din 5 echipe au fost impotriva: "va incetini development-ul",
"sunt prea multe erori de rezolvat".

TASK:
Trebuia sa conving echipele ca beneficiile pe termen lung
depasesc costul pe termen scurt.

ACTION:
- NU am impus decizia top-down
- Am analizat bug-urile din ultimele 6 luni: 34% ar fi fost
  prevenite de strict mode
- Am facut un pilot cu echipa mea: am activat strict mode
  incremental (strictNullChecks prima)
- Am masurat: 2 zile de fix-uri, apoi zero bug-uri de tip
  "cannot read property of undefined" in urmatoarele 2 luni
- Am prezentat rezultatele la engineering all-hands
- Am oferit suport: pair programming sessions pentru echipele
  care voiau sa incerce
- Am creat un tool automat care fixa 60% din erorile de strict mode

RESULT:
- Dupa 2 luni, 4 din 5 echipe au adoptat strict mode voluntar
- A 5-a echipa a adoptat dupa 4 luni
- Bug-urile de tip "null/undefined" au scazut cu 72% in organizatie
- Tool-ul automat a fost open-source-at si folosit de alte companii

INVATAMINTE:
"Rezistenta la schimbare nu se combate cu autoritate, ci cu
date si empatie. Oamenii trebuie sa vada beneficiul si sa
simta ca au ales, nu ca li s-a impus."
```

---

#### 4. "Tell me about a time you had to make a decision with incomplete information"

```
SITUATION:
Un vendor care furniza serviciul de autentificare a anuntat
ca va creste pretul cu 300% in 60 de zile. Trebuia sa decidem:
platim sau migram?

TASK:
Ca Principal Engineer, trebuia sa evaluez optiunile tehnice
si sa fac o recomandare in 48 de ore.

ACTION:
- Am identificat 3 optiuni: platim, migram la alt vendor,
  construim in-house
- Pentru fiecare optiune am evaluat: cost, risc, effort, timeline
- NU aveam benchmarks complete — am facut estimari conservative
  si am identificat explicit ce NU stiam
- Am prezentat un "decision matrix" cu confidence levels:
  "Sunt 80% sigur pe effort, 60% sigur pe timeline"
- Am recomandat: migrare la Auth0, cu timeline de 45 zile,
  cu fallback pe vendor-ul actual (platim o luna in plus daca
  intarziem)
- Am identificat riscul principal: "Nu stim cat de complex e
  migration path-ul pentru custom claims" si am alocat un
  PoC de 2 zile pentru a reduce incertitudinea

RESULT:
- Decizia luata in 48 de ore, PoC validat in 2 zile
- Migrarea completata in 38 de zile (sub deadline)
- Cost redus cu 60% fata de noul pret al vendor-ului vechi
- Custom claims migration a fost mai simpla decat anticipat

INVATAMINTE:
"Deciziile cu informatie incompleta necesita: transparenta
despre ce nu stii, planuri de contingenta, si PoC-uri
rapide pentru a reduce incertitudinea pe riscurile critice."
```

---

#### 5. "Describe how you mentored someone"

```
SITUATION:
Alex, un mid-level Angular developer, era tehnic bun dar
evita orice task care implica arhitectura sau comunicare
cross-team. Managerul lui mi-a cerut sa il mentorizez.

TASK:
Sa il ajut pe Alex sa creasca de la mid-level la senior
in 9 luni.

ACTION:
- Am inceput cu 1-on-1 saptamanal (30 min)
- Luna 1-2: L-am inclus in design review-uri ca observator
  (sa vada cum se discuta arhitectura)
- Luna 3-4: I-am dat un feature de complexitate medie si
  l-am rugat sa propuna arhitectura inainte de implementare.
  Am facut pair programming pe partile dificile.
- Luna 5-6: L-am incurajat sa scrie primul ADR. Am facut
  review impreuna, iterand de 3 ori.
- Luna 7-8: L-am rugat sa mentorizeze un junior pe un task
  specific (teaching as learning)
- Luna 9: L-am nominalizat sa prezinte la tech talk intern

RESULT:
- Alex a fost promovat la Senior dupa 10 luni
- A scris 3 ADR-uri adoptate de echipa
- A devenit mentorul a 2 juniori
- Feedback-ul lui: "Cel mai valoros lucru a fost ca nu mi-ai
  dat raspunsuri direct, ci m-ai lasat sa le gasesc singur
  cu suport cand ma blocam"

INVATAMINTE:
"Cel mai bun mentoring nu e sa dai raspunsuri, ci sa creezi
oportunitati de crestere graduala si sa oferi un spatiu sigur
pentru greseli."
```

---

#### 6. "How did you handle a critical production incident"

```
SITUATION:
Vineri la 17:00, aplicatia principala (Angular + API) a
inceput sa returneze erori 500 pentru 30% din utilizatori.
Revenue impact: ~50K EUR/ora.

TASK:
Ca Principal Engineer si incident commander, trebuia sa
coordonez rezolvarea.

ACTION:
- Minut 0-5: Am declarat incident, am deschis war room
  (Slack channel + call), am anuntat stakeholders
- Minut 5-15: Am triajat — 500-urile veneau de la un singur
  endpoint (product search). Am verificat: deployment recent?
  Da, acum 45 min.
- Minut 15-20: Am decis rollback la versiunea anterioara.
  In paralel, am pus pe cineva sa analizeze diff-ul deployment-ului.
- Minut 20-30: Rollback complet. Erori rezolvate. Impact
  total: 30 min.
- Dupa incident: Am identificat cauza root — un query N+1
  introdus in deployment care sub load depasea timeout-ul DB.
  Code review-ul nu il detectase.
- Am scris post-mortem cu 3 action items:
  1. Adaugat load testing in CI pipeline
  2. Creat alerting pe query duration > 500ms
  3. Training echipei pe N+1 detection

RESULT:
- Downtime total: 30 minute (SLA target: <60 min)
- Action items implementate in urmatorul sprint
- Zero incidente similare in urmatoarele 6 luni
- Post-mortem-ul a devenit template pentru organizatie

INVATAMINTE:
"In incidente, viteza de decizie e mai importanta decat
decizia perfecta. Rollback first, investigate later.
Si cel mai valoros output nu e fix-ul, ci prevenirea:
action items-urile din post-mortem."
```

---

#### 7. "Tell me about a time you had to simplify a complex system"

```
SITUATION:
Aplicatia Angular avea un sistem de formulare custom cu 15
directive, 8 servicii si un DSL (Domain Specific Language)
propriu. Niciun developer nou nu il intelegea fara 2 saptamani
de training.

TASK:
Sa simplific sistemul fara sa pierd functionalitatea si fara
downtime.

ACTION:
- Am cartografiat toate use case-urile: 47 formulare, dar doar
  5 pattern-uri distincte
- Am identificat ca 80% din complexitate servea 20% din cazuri
  (edge cases)
- Am propus inlocuirea cu Angular Reactive Forms + o librarie
  mica de utilities (500 linii vs 5000 linii original)
- Am migrat incremental folosind "strangler fig pattern":
  noul sistem coexista cu vechiul
- Am migrat un formular pe saptamana, cu teste
- Am documentat fiecare pattern intr-un "Form Cookbook"

RESULT:
- De la 5000 linii la 500 linii (-90%)
- Onboarding time de la 2 saptamani la 2 ore
- Bug-uri in formulare: -65%
- Developer satisfaction (survey): de la 2.1 la 4.3 / 5
- Edge case-urile (20%) au fost rezolvate cu configuratie,
  nu cu cod custom

INVATAMINTE:
"Simplificarea nu inseamna sa stergi features, ci sa le
reorganizezi. Intotdeauna intreaba: 'Care sunt cele 5
pattern-uri care acopera 80% din cazuri?' si optimizeaza
pentru alea."
```

---

#### 8. "How did you influence a decision you didn't have authority over"

```
SITUATION:
CTO-ul a decis ca toate echipele vor folosi React pentru
proiectele noi. Echipa mea (si alte 3 echipe) avea
expertiza profunda in Angular si proiecte Angular in productie.

TASK:
Sa influentez decizia in favoarea unei abordari mai nuantate,
fara autoritatea de a o schimba direct.

ACTION:
- NU am contestat public. Am cerut un 1-on-1 cu CTO-ul.
- Am inteles motivatia lui: "React are pool de talente mai mare"
- Am pregatit date:
  - Cost de re-training: 4 echipe x 3 luni = ~500K EUR
  - Cost de rescriere a proiectelor existente: estimare 12 luni
  - Hiring data: in piata noastra, Angular si React erau
    echivalente ca pool de candidati
  - Risc: 2 proiecte critice in Angular ar fi instabile
    in timpul migrarii
- Am propus alternativa: "Angular pentru proiectele existente
  si echipele cu expertiza, React permis pentru proiecte noi
  daca echipa prefra. Evaluam peste 12 luni."
- Am aliniat 3 Tech Leads care au sustinut pozitia

RESULT:
- CTO-ul a acceptat abordarea hibrida
- Dupa 12 luni: 2 proiecte noi in React, 8 in Angular
- Hiring nu a fost afectat
- Am evitat ~500K EUR in costuri de migrare inutile

INVATAMINTE:
"Influenta fara autoritate vine din: intelegerea motivatiei
celuilalt, date concrete, propunerea unei alternative care
adreseaza preocuparile ambelor parti, si aliati care
sustin pozitia."
```

---

#### 9. "Describe a time you had to balance technical debt with new features"

```
SITUATION:
Echipa avea 6 luni de tech debt acumulat (teste lipsa,
workaround-uri, dependencies outdated). In acelasi timp,
business-ul avea 4 features critice pentru Q3 cu deadline
ferm.

TASK:
Sa gasesc un echilibru care livreaza features fara sa
agraveze tech debt-ul.

ACTION:
- Am cuantificat tech debt-ul: impact pe velocity, bug rate,
  developer satisfaction
- Am clasificat debt-ul in 3 categorii:
  A) Blocheaza features noi (fix obligatoriu)
  B) Incetineste development (important dar nu urgent)
  C) "Nice to have" (nu afecteaza delivery)
- Am propus regula "20% debt payment":
  - 80% din sprint pe features
  - 20% pe tech debt (focusat pe categoria A si B)
- Am integrat debt fix-ul IN features:
  "Feature X necesita refactoring pe modulul Y oricum.
  Adaugam 2 zile si facem si cleanup-ul."
- Am creat un "Tech Debt Board" vizibil in sprint planning

RESULT:
- 4 din 4 features livrate la timp
- Tech debt redus cu 40% in aceleasi 6 luni
- Bug rate scazut cu 25%
- Velocity echipei a crescut cu 15% in Q4 (datorita
  debt-ului platit)

INVATAMINTE:
"Tech debt nu e opusul features — e fundamentul pe care
features se construiesc. Cel mai eficient mod de a-l plati
e incremental, integrat in munca zilnica, nu ca un proiect
separat pe care business-ul il poate taia."
```

---

#### 10. "Tell me about your biggest technical achievement"

```
SITUATION:
Organizatia avea 12 aplicatii Angular separate, fiecare cu
propriul repository, propria configuratie CI/CD, propriile
versiuni de dependinte. O actualizare Angular dura 3 luni
si implica 12 echipe separate.

TASK:
Sa unific ecosistemul intr-un monorepo cu shared infrastructure,
reducand duplicarea si accelerand delivery.

ACTION:
- Am scris un RFC detaliat de 15 pagini cu analiza cost/benefit
- Am ales Nx ca monorepo tool dupa evaluarea a 4 alternative
- Am creat un plan de migrare in 6 faze (12 saptamani)
- Am construit un "migration toolkit" (schematics custom)
  care automatiza 70% din procesul de migrare
- Am migrat prima aplicatie personal (proof of concept)
- Am format 12 "migration champions" (1 per echipa)
- Am tinut office hours zilnice in primele 4 saptamani
- Am creat un design system shared extras din codul duplicat

RESULT:
- 12 repositories -> 1 monorepo in 10 saptamani (sub estimare)
- Angular update: de la 3 luni la 3 zile
- Build time: de la 15 min la 3 min (affected projects only)
- 340 componente duplicate -> 89 componente shared
- Developer onboarding: de la 2 saptamani la 3 zile
- Estimare cost savings: ~800K EUR/an in development time

INVATAMINTE:
"Cel mai mare achievement nu e tehnic — e organizatoric.
Tooling-ul si codul au fost partea usoara. Partea grea a
fost sa conving 12 echipe sa renounte la autonomia repo-ului
propriu in favoarea unui beneficiu colectiv. Asta a necesitat
date, empatie, suport constant si celebrarea fiecarei echipe
care a migrat."
```

---

## 10. Cum te diferentiezi de un Senior Engineer

### Gandire la nivel de organizatie vs echipa

```
ACEEASI PROBLEMA, DOUA PERSPECTIVE:

Problema: "Aplicatia noastra Angular se incarca lent"

SENIOR ENGINEER:
"Voi optimiza lazy loading-ul si voi reduce bundle size-ul
aplicatiei noastre. Am identificat 3 componente care pot fi
lazy loaded si 2 librarii care pot fi inlocuite cu alternative
mai mici."

PRINCIPAL ENGINEER:
"Am analizat toate cele 8 aplicatii si am descoperit ca 6 din
ele au aceeasi problema. Cauza root e lipsa unui performance
budget standardizat si a unui process de monitoring. Voi:
1. Defini un performance budget pentru toata organizatia
   (LCP < 1.5s, bundle < 500KB)
2. Integra Lighthouse CI in pipeline-ul tuturor aplicatiilor
3. Crea un 'Performance Cookbook' cu pattern-uri standard
4. Organiza un workshop de performance optimization
5. Implementa un dashboard de Core Web Vitals cross-app

Rezultatul: nu doar aplicatia noastra va fi rapida, ci TOATE
aplicatiile vor fi rapide si vor RAMANE rapide."
```

### Proactive Problem Identification

```
SENIOR: Rezolva problemele cand apar (reactiv)
PRINCIPAL: Previne problemele inainte sa apara (proactiv)

EXEMPLE:

1. DEPENDENCY MANAGEMENT
   Senior: "Am actualizat Angular cand a aparut o vulnerabilitate"
   Principal: "Am creat un proces automat de dependency updates
   cu Renovate Bot, care propune update-uri saptamanal cu teste
   automate, inainte ca vulnerabilitatile sa devina critice"

2. SCALABILITY
   Senior: "Am optimizat query-ul cand a devenit lent"
   Principal: "Am implementat load testing automat in CI care
   detecteaza regressions de performance INAINTE de deployment.
   Am definit SLOs si alerts care ne avertizeaza cand ne
   apropiem de limite, nu cand le-am depasit."

3. TEAM HEALTH
   Senior: "Am ajutat colegul cand a cerut ajutor"
   Principal: "Am creat un engineering survey trimestrial care
   masoara developer experience. Am identificat ca build time-ul
   era frustration #1 si am initiat un proiect de optimizare
   INAINTE ca oamenii sa inceapa sa plece."
```

### Multiplier Effect

```
SENIOR ENGINEER = ADITIV
Output-ul organizatiei creste cu cat produce acea persoana.
Daca senior-ul scrie 100 unitati de cod pe saptamana,
organizatia produce cu 100 mai mult.

PRINCIPAL ENGINEER = MULTIPLICATOR
Output-ul organizatiei se multiplica prin actiunile acelei persoane.

EXEMPLE DE EFECT MULTIPLICATOR:

1. TOOLING
   "Am creat un Nx generator care genereaza un feature complet
   (component, service, store, tests) in 30 secunde.
   Inainte, setup-ul manual lua 2 ore.
   50 developeri x 2 features/saptamana x 2 ore economie =
   200 ore/saptamana economisire = 5 FTE echivalent"

2. STANDARDS
   "Am definit un coding standard care elimina 80% din
   comentariile de code review. Review time: de la 45 min
   la 15 min per PR.
   50 developeri x 5 PR-uri/saptamana x 30 min economie =
   125 ore/saptamana = 3 FTE echivalent"

3. MENTORING
   "Am mentorat 5 mid-levels care au devenit seniors.
   Acum ei mentoreaza fiecare cate 2-3 juniori.
   Impact cascadat: 5 -> 15 persoane imbunatatite"

4. ARCHITECTURE
   "Am definit o arhitectura care reduce complexitatea
   feature-urilor noi cu 40%.
   Fiecare developer livreaza 40% mai repede = multiplicator
   de 1.4x pe toata organizatia"
```

### Business Acumen si Strategic Thinking

```
SENIOR: "Trebuie sa migram la signals pentru ca e mai performant"
PRINCIPAL: "Migrarea la signals va reduce development time cu 20%,
ceea ce ne permite sa livram 2 features in plus in Q3, ceea ce
va genera ~200K EUR revenue aditional conform estimarilor PM-ului"

CUM DEZVOLTI BUSINESS ACUMEN:
1. Participa la product planning meetings
2. Intelege modelul de business (cum face compania bani)
3. Cunoaste metrici business: ARR, churn, CAC, LTV
4. Intreaba PM-ul: "Care e impactul financiar al acestui feature?"
5. Citeste quarterly business reviews
6. Intelege competitia si piata

FRAMEWORK DE PRIORITIZARE TEHNICA (cu business lens):

| Proiect tehnic | Efort | Impact tehnic | Impact business | Prioritate |
|----------------|-------|---------------|-----------------|------------|
| Migrare signals | 3 saptamani | Performance +30% | Conversion +2% | ALTA |
| Design system | 8 saptamani | DX +50% | Time to market -40% | ALTA |
| Rescrierea auth | 6 saptamani | Security +100% | Risk reduction | ALTA |
| Migrare la Bun | 2 saptamani | Build speed +50% | Dev happiness | MEDIE |
| Rescrierea CSS | 4 saptamani | Maintainability | Minimal direct | SCAZUTA |
```

### Long-term Vision vs Short-term Execution

```
SENIOR: Gandeste in sprints (2 saptamani)
PRINCIPAL: Gandeste in quarters si ani

EXEMPLU CONCRET:

Sprint-ul curent: "Feature X trebuie livrat"

Gandirea Senior:
"Implementez Feature X cu BehaviorSubject. E rapid si stiu
cum sa fac. Done in 3 zile."

Gandirea Principal:
"Feature X e primul din 5 features similare care vor veni in
urmatoarele 6 luni. Daca implementez primul cu BehaviorSubject,
urmatoarele 4 vor fi implementate la fel (copy-paste). DAR daca
investesc 2 zile in plus acum sa creez un pattern standard cu
Signal Store, urmatoarele 4 features se vor implementa in
jumatate din timp. ROI: investesc 2 zile acum, economisesc
10 zile pe urmatoarele 6 luni."

Vizualizare:
Fara investitie: [3z][3z][3z][3z][3z] = 15 zile total
Cu investitie:   [5z][1.5z][1.5z][1.5z][1.5z] = 11 zile total
                  ^-- investitie initiala
```

### Comunicare la toate nivelurile

```
SENIOR: Comunica tehnic cu echipa
PRINCIPAL: Adapteaza mesajul la audienta

EXEMPLU: Aceeasi informatie, 4 audiente

CATRE JUNIOR DEVELOPER:
"Voi face un workshop de 2 ore pe change detection in Angular.
Vom invata cum functioneaza Zone.js, de ce signals sunt mai
eficiente, si vom face exercitii practice. Te ajuta sa
intelegi de ce componentele tale se re-rendeaza inutil."

CATRE SENIOR DEVELOPER:
"Migram de la Zone.js la signals-based change detection.
Impactul pe codul tau: inlocuiesti BehaviorSubject cu signal(),
computed() si effect(). Avem un codemod care face 70% automat.
Restul e manual — estimare 1 zi per feature module."

CATRE ENGINEERING MANAGER:
"Migrarea la signals va dura 3 sprints, distribuite pe 2 echipe.
Zero impact pe feature delivery — se face in paralel cu 20%
allocation. Rezultat: aplicatii cu 30% mai responsive, mai
putine bug-uri de rendering, si aliniere cu Angular roadmap."

CATRE VP/CTO:
"Investim 6 saptamani de efort (2 echipe, part-time) pentru a
moderniza rendering engine-ul aplicatiilor. ROI: 30% improvement
in user experience metrics, reducerea riscului tehnic (Angular
va depreca tehnologia actuala), si 20% reducere in bug reports
legate de UI."
```

### Exemple de raspunsuri care demonstreaza gandire de Principal

```
INTREBARE: "Cum ai aborda performance optimization?"

RASPUNS NIVEL SENIOR:
"As folosi Chrome DevTools si Angular DevTools sa identific
componentele lente. As aplica OnPush, trackBy, lazy loading
si as optimiza bundle-ul cu tree shaking."

RASPUNS NIVEL PRINCIPAL:
"As incepe prin a defini ce inseamna 'performant' pentru
business-ul nostru — un SLA cu metrici concrete bazat pe
Core Web Vitals. Apoi:

1. MEASURE: As implementa Real User Monitoring (RUM) pe
   toate aplicatiile, nu doar synthetic tests
2. STANDARDIZE: As crea un Performance Budget enforced in CI
   (LCP < 1.5s, bundle < 500KB, CLS < 0.1)
3. EDUCATE: Workshop de performance patterns pentru toate
   echipele, nu doar echipa mea
4. PREVENT: ESLint rules care previn anti-pattern-uri comune
   (subscriptions in templates, heavy computation in getters)
5. MONITOR: Dashboard shared vizibil in office, cu trends
6. ITERATE: Review trimestrial al performance metrics cu
   ajustarea bugetului

Scopul nu e sa optimizez O aplicatie, ci sa creez un SISTEM
in care toate aplicatiile sunt si raman performante."

INTREBARE: "Ce ai face daca doi membri ai echipei nu sunt
de acord pe o abordare tehnica?"

RASPUNS NIVEL SENIOR:
"I-as asculta pe amandoi, as analiza argumentele si as lua
o decizie bazata pe meritele tehnice."

RASPUNS NIVEL PRINCIPAL:
"Mai intai as evalua daca dezacordul e productiv sau blocant.
Dezacordurile productiv genereaza solutii mai bune — nu le
opresc prematur.

Daca e blocant:
1. Separ persoana de pozitie: 'Avem Abordarea A si Abordarea B'
2. Cer fiecaruia sa prezinte cel mai bun argument PENTRU
   abordarea celuilalt
3. Definim criteriile de decizie INAINTE de a evalua:
   performance? maintainability? time to market?
4. Daca datele nu sunt clare: PoC time-boxed de 2 zile
5. Daca datele sunt clare: decide pe baza criteriilor
6. Documentez decizia in ADR cu rationale
7. 'Disagree and commit' — ambii se angajeaza 100%

Pe termen lung, daca acelasi tip de dezacord apare repetat,
creez un standard care il previne: 'Cand avem scenariul X,
folosim Abordarea Y, conform ADR-17.'"
```

---

## Intrebari frecvente de interviu

### 1. "Cum definesti succesul ca Principal Engineer?"

**Raspuns:**
Succesul unui Principal Engineer nu se masoara in linii de cod scrise, ci in **impactul multiplicator** pe care il are asupra organizatiei. Concret:

- **Viteza echipelor creste**: time-to-market pentru features noi scade constant
- **Calitatea se imbunatateste**: bug rate-ul si incidentele de productie scad
- **Oamenii cresc**: mid-levels devin seniors, seniors devin staff engineers
- **Deciziile tehnice sunt bune**: putine regrete, putine "rescriieri din scratch"
- **Alinierea exista**: echipele nu reinventeaza roata, standards sunt respectate voluntar

Un indicator subiectiv: daca pleci in vacanta 2 saptamani si totul merge bine, inseamna ca ai construit sistemele si cultura corecte. Daca totul se opreste cand pleci, ai creat dependinta, nu leadership.

---

### 2. "Cum gestionezi situatia in care CTO-ul vrea o solutie pe care o consideri gresita?"

**Raspuns:**
Pasul 1: Ma asigur ca inteleg **de ce** vrea acea solutie. De multe ori, CTO-ul are context pe care eu nu il am (presiune de business, promisiuni la clienti, constrangeri de buget).

Pasul 2: Pregatesc o analiza obiectiva cu **date**, nu opinii. "Abordarea A costa X, dureaza Y, are riscul Z. Abordarea B (a mea) costa X', dureaza Y', are riscul Z'."

Pasul 3: Propun o **alternativa** care adreseaza preocuparile ambelor parti. De exemplu: "Inteleg ca trebuie livrat in Q2. Ce-ar fi sa facem Abordarea B cu scope redus, care livreaza 80% din valoare in Q2 si completam restul in Q3?"

Pasul 4: Daca CTO-ul insista dupa ce am prezentat datele — aplic **disagree and commit**. Documentez pozitia mea intr-un ADR (pentru referinta viitoare) si ma angajez 100% la implementarea deciziei luate.

Pasul 5: Daca decizia prezinta riscuri de securitate sau compliance reale, escalez formal si documentat.

---

### 3. "Cum masori impactul muncii tale ca Principal Engineer?"

**Raspuns:**
Folosesc metrici pe 4 dimensiuni:

**Delivery (DORA metrics):**
- Deployment frequency: cat de des livram in productie
- Lead time: cat dureaza de la commit la productie
- Change failure rate: ce procent din deployments cauzeaza probleme
- Mean time to recovery: cat de repede rezolvam incidentele

**Quality:**
- Bug rate per feature
- Test coverage trend
- Production incidents severity si frequency
- Tech debt ratio (timp planificat pe debt vs features)

**Developer Experience:**
- Developer satisfaction survey (trimestrial)
- Onboarding time pentru developeri noi
- Build/test time (DX metric)
- Code review turnaround time

**People Growth:**
- Numarul de promotii in echipele pe care le influentez
- Numarul de mentees care au avansat un nivel
- Numarul de ADR-uri/RFC-uri scrise de altii (cultura de documentatie)
- Numarul de tech talks prezentate de alte persoane

---

### 4. "Cum abordezi o codebase legacy pe care nimeni nu vrea sa lucreze?"

**Raspuns:**
Legacy code nu e o problema tehnica — e o problema de **strategie si motivatie**.

**Strategie:**
1. Evaluez impactul: cat costa in bug-uri, development time, morale?
2. Clasificat debt-ul: ce e critic (securitate, stabilitate) vs nice-to-have?
3. Aplic **strangler fig pattern**: nu rescriem, ci inlocuim incremental
4. Fiecare feature nou se construieste in stil "modern", reducand suprafata legacy-ului

**Motivatie:**
1. Fac eu primul refactoring (lead by example)
2. Celebrez fiecare bucata de legacy eliminata
3. Creez un "Legacy Burndown Chart" vizibil pentru toata echipa
4. Aloc 20% din fiecare sprint explicit pentru modernizare
5. Rotez responsabilitatea — nimeni nu lucreaza doar pe legacy

**Comunicare catre business:**
"Nu cerem 3 luni sa facem refactoring. Cerem 20% din fiecare sprint. In 6 luni, code-ul legacy va fi redus cu 40%, bug rate-ul va scadea cu 30%, si development velocity va creste cu 25%."

---

### 5. "Cum construiesti o echipa tehnica puternica?"

**Raspuns:**
O echipa tehnica puternica are 4 ingrediente:

**1. Diversitate de skill-uri:**
Nu toti trebuie sa fie experti Angular. Ai nevoie de persoane bune pe performance, pe testing, pe UX, pe DevOps. Echilibrul conteaza mai mult decat excelenta individuala.

**2. Siguranta psihologica:**
Oamenii trebuie sa poata spune "nu stiu" si "am gresit" fara consecinte negative. Asta se construieste prin exemplu — cand eu (Principal Engineer) recunosc ca am gresit, altii fac la fel.

**3. Autonomie cu aliniere:**
Echipa decide CUM implementeaza (autonomie), dar respecta standardele si directia tehnica a organizatiei (aliniere). Nu micro-management, dar nici haos.

**4. Cultura de feedback continuu:**
Nu asteptam review-urile anuale. Code reviews, 1-on-1 saptamanale, retro-uri, tech talks — toate sunt forme de feedback care imbunatatesc echipa constant.

---

### 6. "Ce faci cand doua echipe au nevoi tehnice conflictuale?"

**Raspuns:**
Conflictele intre echipe sunt adesea un simptom al lipsei de aliniere la nivel organizational. Abordarea mea:

1. **Inteleg nevoile reale** (nu pozitiile declarate). Echipa A zice "vrem GraphQL", dar nevoia reala e "vrem sa reducem over-fetching". Echipa B zice "vrem REST", dar nevoia reala e "vrem simplitate si caching".

2. **Caut solutia care satisface ambele nevoi**: poate un BFF (Backend for Frontend) care expune GraphQL catre frontend dar consuma REST intern.

3. Daca nu exista solutie win-win, prioritizez pe baza **impactului pe business** si pe baza **reversibilitatii deciziei** (prefer decizia mai usor de schimbat).

4. Documentez decizia intr-un ADR cu rationale-ul, astfel incat sa nu reluam discutia peste 3 luni.

---

### 7. "Cum ramai tehnic relevant cand petici mult timp pe leadership?"

**Raspuns:**
E cea mai mare provocare a rolului de Principal Engineer. Strategia mea:

- **20-40% din timp pe cod**: nu renunt la cod complet. Aleg task-uri strategice (PoC-uri, tooling, migration scripts) nu task-uri de pe critical path (nu vreau sa blochez echipa)
- **Code reviews zilnice**: citesc codul altora — ramui la curent cu codebase-ul si cu practicile echipelor
- **Side projects personale**: experimentez cu tehnologii noi in afara orelor de lucru (1-2 ore pe saptamana)
- **Conferinte si comunitate**: urmaresc Angular Blog, participi la conferinte, citesc RFC-urile Angular
- **Pair programming**: periodoc fac pair cu developeri din echipe diferite — invat si eu de la ei
- **"Gardening" sessions**: 2 ore pe saptamana in care fac refactoring, actualizez dependinte, imbunatatesc tooling

---

### 8. "Descrie o situatie in care ai renuntat la propria solutie in favoarea solutiei altcuiva"

**Raspuns framework (STAR):**

**S:** Propusesem o arhitectura de micro-frontends bazata pe Module Federation pentru reorganizarea aplicatiei noastre monolitice.

**T:** Un senior engineer din alta echipa a propus o alternativa bazata pe o arhitectura monorepo cu Nx si lazy-loaded feature libraries — fara micro-frontends.

**A:** Initial am fost skeptic pentru ca investisem mult timp in propunerea mea. Dar am ascultat argumentele lui:
- Complexitate mult mai mica (nu trebuie shared dependencies management)
- Deployment unificat (nu trebuie orchestrare multi-app)
- Echipa nu avea experienta cu micro-frontends (curba de invatare)
Am facut un PoC time-boxed de 3 zile pentru ambele abordari. Rezultatele au aratat ca monorepo-ul atingea 90% din beneficiile micro-frontends cu 30% din complexitate.

**R:** Am sustinut public solutia colegului in design review. Echipa a adoptat-o si proiectul s-a completat cu 4 saptamani mai devreme decat estimarea mea initiala. Colegul a crescut enorm in incredere si vizibilitate. Am invatat ca rolul meu nu e sa am dreptate, ci sa ma asigur ca organizatia ia cea mai buna decizie — indiferent cine o propune.

---

### 9. "Cum gestionezi burnout-ul in echipa?"

**Raspuns:**
Burnout-ul e un simptom, nu o cauza. Ca Principal Engineer:

**Detectie:**
- Urmaresc metrici de engineering health: overtime hours, weekend commits, sprint burndown velocity drops
- In 1-on-1: intreb direct "Cum te simti? Pe o scara de la 1 la 10, cat de sustenabil e ritmul actual?"
- Observ semnale: cineva care era activ in discutii devine tacut, code review quality scade, cinicismul creste

**Prevenire:**
- Protejez echipa de scope creep si "urgente" artificiale
- Ma asigur ca sprint-urile sunt realiste (nu overcommit)
- Aloc timp explicit pentru learning si experimentare
- Sarbatoresc livrarea, nu overtime-ul

**Interventie:**
- Vorbesc 1-on-1 cu persoana afectata
- Reduc temporar workload-ul (redistribui task-uri)
- Identific cauza root: prea multa munca? munca neinteresanta? lipsa de control? lipsa de recunoastere?
- Adresez cauza, nu simptomul
- Escalez catre manager daca e o problema sistemica

---

### 10. "Ce intrebari ai pentru noi?" (intrebari pe care TU le pui la interviu)

**Intrebari care demonstreaza gandire de Principal:**

1. "Care e cea mai mare provocare tehnica pe care o are organizatia in urmatoarele 12 luni?"
   *(Arata ca gandesti strategic si pe termen lung)*

2. "Cum arata procesul de decizie tehnica? Aveti ADR-uri sau RFC-uri?"
   *(Arata ca iti pasa de governance si documentatie)*

3. "Care e raportul dintre tech debt si feature work? Cum se negociaza?"
   *(Arata ca intelegi tensiunea reala din orice organizatie)*

4. "Cum arata cariera de IC (Individual Contributor) in organizatie? Exista un track pana la Distinguished Engineer?"
   *(Arata ca gandesti pe termen lung si ca vrei sa cresti)*

5. "Care e cel mai mare regret tehnic din ultimii 2 ani? Ce ati invatat?"
   *(Arata maturitate — nu te astepti la perfectiune)*

6. "Cat de autonome sunt echipele in a-si alege tool-urile si abordarea tehnica?"
   *(Arata ca iti pasa de cultura si de balanta autonomie/aliniere)*

7. "Cum masarati developer experience si developer productivity?"
   *(Arata ca iti pasa de oameni, nu doar de output)*

8. "Ce rol joaca Principal Engineer-ul in hiring si in definirea culturii tehnice?"
   *(Arata ca intelegi scope-ul rolului dincolo de cod)*
