# 10. AI in Dezvoltare Software

## Cuprins
- [10.1 Unelte AI actuale](#101-unelte-ai-actuale)
- [10.2 Cum sa folosesti AI eficient in development](#102-cum-sa-folosesti-ai-eficient-in-development)
- [10.3 AI in interviuri tehnice](#103-ai-in-interviuri-tehnice)
- [10.4 Prompt engineering pentru cod](#104-prompt-engineering-pentru-cod)
- [10.5 Evaluarea si verificarea codului generat de AI](#105-evaluarea-si-verificarea-codului-generat-de-ai)
- [10.6 Riscuri si limitari ale AI](#106-riscuri-si-limitari-ale-ai)
- [10.7 Cum AI schimba rolul de Principal Engineer](#107-cum-ai-schimba-rolul-de-principal-engineer)
- [10.8 Best practices pentru integrarea AI in workflow](#108-best-practices-pentru-integrarea-ai-in-workflow)
- [Intrebari frecvente de interviu](#intrebari-frecvente-de-interviu)

---

## 10.1 Unelte AI actuale

### GitHub Copilot

GitHub Copilot este cel mai raspandit tool AI pentru dezvoltare software, construit pe modele de limbaj (initial Codex, acum GPT-4 si Claude) si integrat direct in editoare.

**Cum functioneaza:**
- Analizeaza codul din fisierul curent, fisierele deschise si contextul proiectului
- Foloseste un model LLM (Large Language Model) pentru a prezice urmatoarea secventa de cod
- Trimite contextul la server (cloud-based), primeste completari in timp real
- Modelul a fost antrenat pe cod public de pe GitHub (ceea ce ridica si probleme de licenta)

**Moduri de utilizare:**

1. **Inline completions** - sugestii automate in timp ce scrii cod
   - Apare ca text gri (ghost text) pe care il accepti cu Tab
   - Functioneaza cel mai bine cand ai un pattern clar sau un comentariu descriptiv
   - Exemplu: scrii `// funcție care validează email` si Copilot genereaza implementarea

2. **Chat mode** - conversatie cu AI despre cod
   - Panel lateral in VS Code unde pui intrebari
   - Poate vedea fisierul curent si fisierele selectate
   - Util pentru explicatii, debugging, refactoring
   - Suporta comenzi precum `/explain`, `/fix`, `/tests`

3. **Agent mode** (Copilot Workspace / Copilot Agent)
   - Poate executa task-uri complexe multi-step
   - Creeaza plan, modifica mai multe fisiere, ruleaza comenzi terminal
   - Poate rula teste si itera pana trec
   - Cel mai aproape de un "coleg AI" care lucreaza autonom

**In context Angular:**
```typescript
// Copilot exceleaza la generarea de boilerplate Angular
// Scrii comentariul, Copilot genereaza componenta:

// Component that displays a paginated table of users with sorting
@Component({
  selector: 'app-user-table',
  // Copilot va sugera template, styles, imports...
})
export class UserTableComponent implements OnInit {
  // Copilot va sugera proprietatile si metodele bazate pe context
}
```

### Claude Code (Anthropic)

Claude Code este un CLI tool agentic dezvoltat de Anthropic, care ruleaza direct in terminal.

**Caracteristici cheie:**
- **CLI-native** - functioneaza din terminal, nu necesita editor specific
- **Agentic capabilities** - poate citi/scrie fisiere, rula comenzi, naviga codebase
- **Context mare** - fereastra de context de 200K tokens, poate intelege proiecte intregi
- **Tool use** - are acces la filesystem, shell, grep, glob si alte utilitare
- **Iterativ** - poate rula teste, vedea erorile, si corecta automat

**Workflow tipic cu Claude Code:**
```bash
# Deschizi Claude Code in directorul proiectului
claude

# Descrii ce vrei
> "Adauga lazy loading pentru modulul de admin cu guards si
>  redirect la login daca userul nu e autentificat"

# Claude Code:
# 1. Citeste structura proiectului
# 2. Identifica fisierele relevante
# 3. Propune si aplica modificarile
# 4. Ruleaza testele sa verifice
```

**CLAUDE.md files:**
- Fisier de configurare plasat in root-ul proiectului
- Defineste conventii, stack tehnic, reguli specifice
- Claude Code il citeste automat si respecta conventiile echipei
- Echivalentul unui "onboarding document" pentru AI

### Cursor

Cursor este un editor de cod construit de la zero cu AI ca prioritate (fork de VS Code).

**Moduri de utilizare:**

1. **Tab completion** - similar cu Copilot dar cu model propriu
   - Predictii mai agresive, poate sugera blocuri intregi de cod
   - "Next action prediction" - anticipeaza ce vrei sa faci

2. **Chat (Cmd+L)** - conversatie cu context automat
   - Vede automat fisierele relevante din proiect
   - Poate vedea erori din terminal, linter, TypeScript compiler
   - Raspunde cu cod aplicabil direct cu un click

3. **Compose (Cmd+I)** - editare inline
   - Selectezi cod, descrii modificarea
   - AI-ul modifica direct in fisier cu diff vizibil
   - Accept/reject per modificare

4. **Multi-file editing** - agent mode
   - Poate modifica mai multe fisiere simultan
   - Creeaza fisiere noi, sterge cod, refactoreaza
   - "Apply all" pentru a aplica toate modificarile

5. **@-references** - context explicit
   - `@file` pentru a include un fisier specific
   - `@codebase` pentru search semantic in tot proiectul
   - `@docs` pentru documentatie externa
   - `@web` pentru cautare pe internet

### Alte unelte notabile

**Windsurf (Codeium)**
- Editor AI similar cu Cursor (tot fork VS Code)
- "Cascade" - flow agentic care mentine context intre actiuni
- Accent pe intelegerea intregului proiect (codebase-aware)
- Free tier generos, alternativa accesibila la Cursor

**Cline (VS Code Extension)**
- Extension open-source pentru VS Code
- Complet agentic - poate crea fisiere, rula comenzi, instala dependinte
- Suporta orice provider LLM (OpenAI, Anthropic, local models)
- Transparent - arata fiecare actiune si cere aprobare
- Costul este direct pe API usage (nu subscription)

**Aider**
- CLI tool open-source pentru pair programming cu AI
- Integrat cu Git - fiecare modificare devine un commit
- Suporta multiple modele (GPT-4, Claude, local models)
- Excelent pentru proiecte existente - intelege repo maps
- "Architect mode" - un model planifica, altul implementeaza

**Amazon Q Developer (fost CodeWhisperer)**
- Integrat in ecosistemul AWS
- Bun pentru cod AWS/cloud (Lambda, CDK, CloudFormation)
- Security scanning incorporat
- Free tier pentru uz individual

### Comparatie: abordari AI in dezvoltare

| Abordare | Descriere | Exemplu | Cand e util |
|----------|-----------|---------|-------------|
| **Inline completion** | Sugestii automate in timp ce scrii | Copilot Tab, Cursor Tab | Boilerplate, pattern-uri repetitive, cod predictibil |
| **Chat** | Conversatie Q&A despre cod | Copilot Chat, Cursor Chat | Debugging, explicatii, intrebari punctuale |
| **Inline edit** | Modificare directa in fisier | Cursor Compose, Copilot Edits | Refactoring, fix-uri rapide, transformari de cod |
| **Agentic** | Executie autonoma multi-step | Claude Code, Copilot Agent, Cline | Task-uri complexe, features noi, migrari |

**Evolutia abordarii:**
- **2022**: Inline completions (Copilot original) - "autocomplete pe steroizi"
- **2023**: Chat mode - "intreaba AI-ul despre cod"
- **2024**: Agentic mode - "AI-ul executa task-uri"
- **2025-2026**: Agentic autonom - "AI-ul lucreaza independent pe task-uri definite"

**Tendinta clara:** Se trece de la "AI ca autocomplete" la "AI ca agent autonom" care poate planifica, implementa, testa si itera. Rolul developerului evolueaza spre cel de reviewer si director.

---

## 10.2 Cum sa folosesti AI eficient in development

### Code generation - boilerplate si pattern-uri repetitive

AI exceleaza la generarea de cod repetitiv si predictibil. In Angular, aceasta acoperire este semnificativa.

**Exemple unde AI straluceste:**

```typescript
// 1. Generare de services CRUD complete
// Prompt: "Create an Angular service for User CRUD operations
// using HttpClient with error handling and loading state"

@Injectable({ providedIn: 'root' })
export class UserService {
  private apiUrl = '/api/users';

  constructor(private http: HttpClient) {}

  getAll(): Observable<User[]> {
    return this.http.get<User[]>(this.apiUrl).pipe(
      catchError(this.handleError('getAll', []))
    );
  }

  getById(id: string): Observable<User> {
    return this.http.get<User>(`${this.apiUrl}/${id}`).pipe(
      catchError(this.handleError('getById'))
    );
  }
  // ... AI genereaza toate operatiile CRUD corect
}

// 2. Generare de interfaces din API response
// Dai AI-ului un JSON response si primesti TypeScript interfaces

// 3. Reactive forms cu validari complexe
// Descrii campurile si regulile, AI genereaza FormGroup complet

// 4. NgRx boilerplate (actions, reducers, effects, selectors)
// Cel mai tedios boilerplate din Angular - perfect pentru AI
```

### Code review si refactoring assistance

**AI ca reviewer:**
- Paste-uiesti o functie si intrebi "review this code for potential issues"
- AI identifica: probleme de performance, memory leaks, edge cases
- Poate sugera imbunatatiri de readability si maintainability
- Util ca "second pair of eyes" inainte de code review-ul uman

**Refactoring patterns:**
```typescript
// Exemplu: Refactoring de la imperative la reactive
// Inainte (dai acest cod AI-ului):
export class DashboardComponent implements OnInit {
  users: User[] = [];
  loading = false;
  error: string | null = null;

  ngOnInit() {
    this.loading = true;
    this.userService.getAll().subscribe(
      users => { this.users = users; this.loading = false; },
      err => { this.error = err.message; this.loading = false; }
    );
  }
}

// AI poate transforma in varianta reactive cu signals:
export class DashboardComponent {
  private usersResource = rxResource({
    loader: () => this.userService.getAll()
  });

  users = this.usersResource.value;
  loading = this.usersResource.isLoading;
  error = this.usersResource.error;

  constructor(private userService: UserService) {}
}
```

### Scrierea de teste (particularly useful)

**Acesta este poate cel mai bun use case pentru AI in Angular.** Testele sunt repetitive, urmeaza pattern-uri clare, si sunt adesea neglijate din lipsa de timp.

```typescript
// Prompt: "Write comprehensive unit tests for this component"
// AI genereaza:

describe('UserTableComponent', () => {
  let component: UserTableComponent;
  let fixture: ComponentFixture<UserTableComponent>;
  let userService: jest.Mocked<UserService>;

  beforeEach(async () => {
    const mockUserService = {
      getAll: jest.fn(),
      delete: jest.fn()
    };

    await TestBed.configureTestingModule({
      imports: [UserTableComponent],
      providers: [
        { provide: UserService, useValue: mockUserService }
      ]
    }).compileComponents();

    fixture = TestBed.createComponent(UserTableComponent);
    component = fixture.componentInstance;
    userService = TestBed.inject(UserService) as jest.Mocked<UserService>;
  });

  it('should display users when loaded', () => {
    userService.getAll.mockReturnValue(of(mockUsers));
    fixture.detectChanges();
    const rows = fixture.nativeElement.querySelectorAll('tr');
    expect(rows.length).toBe(mockUsers.length + 1); // +1 for header
  });

  it('should show loading spinner while fetching', () => {
    userService.getAll.mockReturnValue(NEVER);
    fixture.detectChanges();
    expect(fixture.nativeElement.querySelector('.spinner')).toBeTruthy();
  });

  it('should display error message on failure', () => {
    userService.getAll.mockReturnValue(
      throwError(() => new Error('Network error'))
    );
    fixture.detectChanges();
    expect(fixture.nativeElement.textContent).toContain('Network error');
  });

  // AI genereaza 10-15 teste relevante pentru fiecare componenta
});
```

**De ce AI exceleaza la teste:**
- Pattern-urile de testare sunt foarte repetitive
- AI-ul vede codul sursa si stie ce trebuie testat
- Poate genera edge cases pe care le-ai fi ratat
- Reduce bariera psihologica de "nu am chef sa scriu teste"

### Generare de documentatie

- JSDoc/TSDoc comments pentru functii si clase
- README files pentru librarii si module
- API documentation din cod
- Changelogs din git history
- Migration guides intre versiuni

### Debugging assistance

```typescript
// Paste-uiesti eroarea si codul relevant:
// "Getting ExpressionChangedAfterItHasBeenCheckedError in this component"

// AI explica:
// 1. De ce apare (change detection cycle deja terminat)
// 2. Cauza comuna (modificare de state in lifecycle hook gresit)
// 3. Solutia concreta pentru cazul tau
// 4. Best practice pentru a evita pe viitor
```

### Learning new APIs si frameworks

- Ceri explicatii cu exemple concrete: "Explain Angular signals with practical examples"
- Compari abordari: "What's the difference between BehaviorSubject and signal?"
- Migration patterns: "How to migrate from NgModules to standalone components?"
- AI-ul poate genera mini-proiecte de invatare

### Ce face AI bine vs ce nu face bine

**AI exceleaza la:**
- Boilerplate si cod repetitiv
- Traducerea intre pattern-uri (imperative -> reactive)
- Generare de teste pentru cod existent
- Explicarea codului si erorilor
- Completare de pattern-uri (daca ai 3 functii similare, genereaza a 4-a)
- Refactoring mecanic (rename, extract, restructure)
- Generare de types/interfaces din exemple

**AI are dificultati cu:**
- Decizii arhitecturale complexe (trade-offs, context business)
- Optimizare de performanta non-trivala (necesita profiling real)
- Business logic specifica domeniului
- Integrari complexe intre sisteme
- Debugging de race conditions si timing issues
- Cod care depinde de state global sau efecte secundare subtile
- Securitate - poate genera cod vulnerabil fara sa semnaleze
- Intelegerea "de ce" nu doar "cum" (motivatia din spatele deciziilor)

---

## 10.3 AI in interviuri tehnice

### Tendinta: companii care permit AI tools in interviuri

**Meta** a fost printre primele companii mari care au anuntat ca permit AI tools in interviurile de coding. Logica: in practica folosesti AI zilnic, de ce sa testezi fara?

**Alte companii care permit sau experimeinteaza:**
- Shopify - Tobi Lutke (CEO) a declarat ca AI este o "baseline expectation"
- Diverse startup-uri tech-forward
- Companiile care au adoptat AI-first culture

**Companiile care inca restrictioneaza:**
- Majoritatea companiilor enterprise traditionale
- Companiile cu procese de interviu rigide (FAANG traditional)
- Contexte unde se testeaza fundamentele (junior roles)

### Cum schimba aceasta ceea ce se testeaza

**Inainte (fara AI):**
- Memorarea de algoritmi si structuri de date
- Viteza de scriere a codului corect
- Cunoasterea sintaxei exacte
- Implementarea de la zero

**Acum (cu AI):**
- **Problem decomposition** - cum descompui o problema complexa in sub-probleme
- **Verification skills** - poti valida daca codul generat de AI e corect?
- **Direction and iteration** - stii sa ghidezi AI-ul spre solutia corecta?
- **Architecture thinking** - poti lua decizii de design pe care AI-ul nu le poate lua?
- **Communication** - poti explica ce faci si de ce?

### Focus pe problem decomposition si verification

```
Exemplu de interviu AI-enabled:

Interviewer: "Design a real-time collaborative document editor"

Candidat (slab):
- "Let me ask Copilot to generate a collaborative editor"
- Accepta primul output fara analiza
- Nu intelege trade-off-urile

Candidat (excelent):
- Descompune problema: conflict resolution, sync protocol,
  offline support, cursor awareness
- Foloseste AI pentru a genera implementarea CRDT
- VERIFICA output-ul: "Acest CRDT handles concurrent deletes corect?"
- Identifica lipsuri: "Lipseste handling-ul pentru offline queue"
- Itereaza: "Adauga optimistic updates cu rollback"
- Explica deciziile: "Am ales CRDT over OT because..."
```

### AI pair programming interviews

Un format nou de interviu:
1. Candidatul primeste un task complex
2. Are acces la AI tools (Copilot, ChatGPT, Claude)
3. Interviewerul observa CUM foloseste AI-ul:
   - Prompturile sunt clare si specifice?
   - Verifica output-ul sau accepta orb?
   - Itereaza eficient?
   - Stie cand AI-ul greseste?
   - Poate explica codul generat?

**Ce impresionanaza:**
- Folosesti AI ca multiplicator, nu ca inlocuitor
- Identifici rapid cand AI-ul hallucineaza
- Stii sa dai context relevant (nu "make it work")
- Combini AI cu expertiza proprie

### Cum sa te pregatesti pentru interviuri AI-enabled

1. **Exerseaza cu AI tools zilnic** - sa devina natural, nu awkward
2. **Invata sa prompt-uiesti eficient** (vezi sectiunea 10.4)
3. **Dezvolta skill-ul de code review** - cel mai important skill
4. **Cunoaste limitarile** - stii cand AI-ul probabil greseste
5. **Practica explicarea** - poti explica fiecare linie de cod generat?
6. **Invata fundamentele** - AI-ul nu te salveaza daca nu intelegi bazele
7. **Fa proiecte reale cu AI** - nu doar exercitii artificiale

---

## 10.4 Prompt engineering pentru cod

### Principii fundamentale

#### 1. Fii specific despre limbaj, framework, versiune

```
// BAD prompt:
"Create a component with a form"

// GOOD prompt:
"Create an Angular 19 standalone component using reactive forms
with signal-based state management. Use the new control flow
syntax (@if, @for) instead of *ngIf/*ngFor.
TypeScript strict mode enabled."
```

#### 2. Ofera context (cod existent, constrangeri)

```
// BAD prompt:
"Add authentication to my app"

// GOOD prompt:
"Add JWT authentication to my Angular 19 app.
Current setup:
- Using HttpClient with interceptors (functional style)
- Router with standalone components and functional guards
- Backend API at /api/auth/login returns { accessToken, refreshToken }
- Need to store tokens in httpOnly cookies (backend sets them)
- Need an AuthService with login(), logout(), isAuthenticated()
- Need a functional HTTP interceptor that adds Authorization header
- Need a functional route guard for protected routes
Here's my current app.config.ts: [paste code]"
```

#### 3. Cere rationament pas cu pas

```
// GOOD prompt:
"I need to implement optimistic updates for a todo list in Angular
with NgRx. Walk me through the approach step by step:
1. First explain the pattern
2. Then show the actions needed
3. Then the reducer logic
4. Then the effect with rollback on failure
5. Then the component integration"
```

#### 4. Foloseste exemple (few-shot prompting)

```
// GOOD prompt:
"I'm creating form validators. Here's the pattern I use:

export function minAge(age: number): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const birthDate = new Date(control.value);
    const today = new Date();
    const diff = today.getFullYear() - birthDate.getFullYear();
    return diff < age ? { minAge: { required: age, actual: diff } } : null;
  };
}

Now create similar validators for:
- dateRange(startControl, endControl) - end must be after start
- uniqueAsync(service, field) - checks backend for uniqueness
- passwordStrength(minScore) - checks complexity"
```

#### 5. Itereaza si rafineaza

```
// Prima iteratie:
"Create a data table component"
// -> Output generic, nu e ce vrei

// A doua iteratie:
"Good start, but I need:
- Virtual scrolling for 10k+ rows (use @angular/cdk)
- Column sorting with multi-column support
- Server-side pagination
- Sticky header
- Row selection with checkbox column
- Responsive: hide columns on mobile"

// A treia iteratie:
"The virtual scrolling implementation has a bug -
when I sort a column, the scroll position resets.
Keep the scroll position after sorting."
```

#### 6. System prompts si CLAUDE.md files

Un fisier `CLAUDE.md` in root-ul proiectului seteaza contextul permanent:

```markdown
# Project Context

## Tech Stack
- Angular 19 with standalone components
- NgRx Signal Store for state management
- Tailwind CSS 4.0 for styling
- Jest for unit tests, Playwright for E2E
- Nx monorepo with multiple apps and shared libs

## Conventions
- Use signals over BehaviorSubjects
- Use new control flow (@if, @for, @switch)
- Functional guards and interceptors (no class-based)
- All components must be OnPush change detection
- Strict TypeScript: no any, no implicit returns
- File naming: feature-name.component.ts (kebab-case)
- Test files colocated: feature-name.component.spec.ts

## Architecture
- Feature-based folder structure
- Shared UI components in libs/ui
- Each feature has: components/, services/, models/, state/
- API calls only in services, never in components
- Smart/container components vs dumb/presentational components

## Common Patterns
- Use rxResource() or httpResource() for data fetching
- Error handling via global ErrorHandler + toast notifications
- Forms use ControlValueAccessor for custom inputs
- Lazy loading per feature route
```

### Exemple practice: good vs bad prompts pentru Angular

**Exemplul 1: Crearea unui service**

```
// BAD:
"Make a service for users"

// GOOD:
"Create an Angular UserService with these requirements:
- Injectable in root
- Methods: getUsers(params: PageParams), getUserById(id: string),
  createUser(dto: CreateUserDto), updateUser(id: string, dto: UpdateUserDto),
  deleteUser(id: string)
- Use HttpClient with proper typing
- Base URL from environment.apiUrl
- Error handling: catch HTTP errors, map to domain errors
- Return Observable for each method
- Include JSDoc comments
- Here are my existing types: [paste interfaces]"
```

**Exemplul 2: Debugging**

```
// BAD:
"My component doesn't work"

// GOOD:
"My Angular component throws 'NG0100: ExpressionChangedAfterItHasBeenChecked'
in development mode. It happens when I navigate to the dashboard route.

Component code: [paste component]
Template: [paste template]
Parent component: [paste relevant parent code]

The error points to line 15 in the template where I bind [data]="computedData".
computedData is a getter that filters an array.

How do I fix this while keeping the reactive approach?"
```

**Exemplul 3: Refactoring**

```
// BAD:
"Refactor this code"

// GOOD:
"Refactor this Angular component from class-based to modern patterns:
1. Convert NgModule component to standalone
2. Replace *ngIf/*ngFor with @if/@for control flow
3. Replace BehaviorSubjects with signals
4. Replace subscribe() calls with toSignal() or rxResource()
5. Add OnPush change detection
6. Keep the same public API (inputs/outputs)
7. Show me the diff, not the full file

Current code:
[paste component code]"
```

---

## 10.5 Evaluarea si verificarea codului generat de AI

### Regula de aur: Always review generated code

**Nu exista "trust but verify" cu AI-generated code. Este "don't trust, always verify."**

Codul generat de AI arata convingator - este bine formatat, urmeaza conventii, si pare complet. Dar aceasta incredere superficiala este exact ceea ce il face periculos.

### Ce sa verifici

#### Security issues

```typescript
// AI poate genera:
const query = `SELECT * FROM users WHERE email = '${email}'`;
// SQL injection vulnerabil!

// Sau in Angular:
this.el.nativeElement.innerHTML = userInput;
// XSS vulnerabil!

// Sau:
localStorage.setItem('authToken', token);
// Tokens nu ar trebui in localStorage (accesibil din JS -> XSS)
```

**Checklist securitate:**
- Input validation prezenta si corecta?
- Output encoding/sanitization?
- Authentication si authorization verificate?
- Secrets hardcoded in cod?
- CORS configurat corect?
- CSP headers considerate?
- Dependinte cu vulnerabilitati cunoscute?

#### Performance

```typescript
// AI genereaza adesea cod corect dar ineficient:

// GENERAT DE AI - functional dar O(n*m):
filterUsers(users: User[], criteria: FilterCriteria): User[] {
  return users.filter(user =>
    criteria.tags.every(tag => user.tags.includes(tag))
  );
}

// OPTIMIZAT - O(n+m) cu Set:
filterUsers(users: User[], criteria: FilterCriteria): User[] {
  const tagSet = new Set(criteria.tags);
  return users.filter(user =>
    user.tags.some(tag => tagSet.has(tag))
  );
}

// AI poate genera si:
// - Subscriptions fara unsubscribe (memory leaks)
// - Change detection fara OnPush
// - Computed values recalculate in fiecare ciclu
// - HTTP calls in template (apelate la fiecare change detection)
```

#### Edge cases

```typescript
// AI genereaza happy path-ul perfect, dar uita edge cases:

// GENERAT DE AI:
calculateAge(birthDate: string): number {
  const birth = new Date(birthDate);
  const today = new Date();
  return today.getFullYear() - birth.getFullYear();
}

// PROBLEME:
// - Ce daca birthDate e null/undefined?
// - Ce daca formatul e invalid?
// - Nu ia in calcul luna si ziua (poate fi cu 1 an off)
// - Ce daca data e in viitor?
// - Timezone issues

// CORECT:
calculateAge(birthDate: string | null | undefined): number | null {
  if (!birthDate) return null;

  const birth = new Date(birthDate);
  if (isNaN(birth.getTime())) return null;

  const today = new Date();
  if (birth > today) return null;

  let age = today.getFullYear() - birth.getFullYear();
  const monthDiff = today.getMonth() - birth.getMonth();
  if (monthDiff < 0 || (monthDiff === 0 && today.getDate() < birth.getDate())) {
    age--;
  }
  return age;
}
```

#### Correctness

- Logica face ce trebuie, nu doar ce pare ca trebuie?
- Algoritmul e corect pentru TOATE input-urile, nu doar exemplele?
- Return types sunt corecte?
- Error handling acopera cazurile reale?

### Ruleaza teste pe codul generat

```bash
# Dupa ce AI genereaza cod, INTOTDEAUNA:

# 1. Ruleaza testele existente (sa nu fi spart nimic)
ng test --watch=false

# 2. Ruleaza linter-ul
ng lint

# 3. Ruleaza build-ul (type checking complet)
ng build

# 4. Ruleaza E2E daca ai
npx playwright test

# 5. Daca AI a generat si teste, verifica ca testele
#    CHIAR testeaza ce trebuie (nu sunt triviale)
```

### Intelege codul inainte de commit

**Regula: Daca nu poti explica fiecare linie, nu dai commit.**

Intrebari sa iti pui:
- Ce face fiecare functie si de ce?
- Care e flow-ul de date?
- Ce se intampla la erori?
- Exista side effects?
- E idiomatic pentru Angular/TypeScript?

### Greseli comune ale AI

**1. Hallucinated APIs:**
```typescript
// AI poate genera:
import { signalStore } from '@ngrx/signals';

export const UserStore = signalStore(
  withState({ users: [] }),
  withComputed(({ users }) => ({
    activeUsers: computed(() => users().filter(u => u.active))
  })),
  withMethods(store => ({
    loadUsers: rxMethod(  // API-ul real poate diferi
      pipe(
        switchMap(() => inject(UserService).getAll()),
        tapResponse(
          users => patchState(store, { users }),
          error => console.error(error)
        )
      )
    )
  }))
);
// Poate folosi API care exista in alta versiune sau nu exista deloc
```

**2. Outdated patterns:**
```typescript
// AI antrenat pe cod vechi poate genera:
@NgModule({
  declarations: [AppComponent],  // Outdated - use standalone
  imports: [BrowserModule],
  bootstrap: [AppComponent]
})
export class AppModule {}

// In loc de (Angular 19 modern):
bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes),
    provideHttpClient(withInterceptors([authInterceptor]))
  ]
});
```

**3. Subtle bugs:**
```typescript
// AI genereaza cod care pare corect dar are bug subtil:
effect(() => {
  const userId = this.selectedUserId();
  // Bug: acest effect se re-ruleaza la ORICE signal change,
  // nu doar cand userId se schimba
  this.loadUserDetails(userId);
});

// Corect: foloseste untracked() sau explicit dependencies
effect(() => {
  const userId = this.selectedUserId();
  untracked(() => {
    this.loadUserDetails(userId);
  });
});
```

### Code review checklist pentru AI-generated code

- [ ] **Compileaza fara erori** - `ng build` trece
- [ ] **Teste trec** - existente + noi
- [ ] **Fara API-uri inexistente** - verifica imports si metode in documentatie
- [ ] **Pattern-uri moderne** - standalone, signals, new control flow
- [ ] **Securitate** - fara XSS, injection, secrets exposed
- [ ] **Performance** - OnPush, fara memory leaks, fara N+1 queries
- [ ] **Edge cases** - null, undefined, empty arrays, erori de retea
- [ ] **Error handling** - nu doar happy path
- [ ] **TypeScript strict** - fara `any`, fara `as` unnecessary
- [ ] **Naming conventions** - urmeaza conventiile proiectului
- [ ] **Intelegi codul** - poti explica fiecare linie

---

## 10.6 Riscuri si limitari ale AI

### Hallucinations (confident but wrong)

AI-ul nu "stie" ce e corect - genereaza text care PARE probabil bazat pe training data. Aceasta inseamna ca poate genera cu aceeasi incredere cod corect si cod complet gresit.

**Exemple in Angular:**
- Inventeaza decoratori care nu exista: `@AutoUnsubscribe()`, `@Cacheable()`
- Sugereaza metode inexistente pe clase Angular reale
- Combina API-uri din versiuni diferite intr-un singur fisier
- Creeaza import paths care nu exista in librarii reale

**De ce e periculos:** Output-ul arata perfect formatat, cu TypeScript types corecte, urmeaza conventii - dar functioneaza cod ce nu exista. Code review-ul uman e singura aparare.

### Outdated training data

Modelele AI sunt antrenate pe date pana la un cutoff date. Chiar si cu acces la web, pot mixa informatii vechi cu noi.

**Impact in Angular (care evolueaza rapid):**
- Angular 19 a introdus `linkedSignal`, `rxResource` - modele vechi nu le cunosc
- Trecerea la standalone components e recenta - AI-ul inca genereaza NgModules
- `@if/@for/@switch` au inlocuit directivele structurale - AI-ul mixeaza ambele
- Signal-based components sunt noi - AI-ul inca prefera Observable-based patterns

**Mitigare:**
- Specifica versiunea in prompt: "Angular 19, standalone components"
- Include exemplu de pattern dorit in prompt
- Verifica in documentatia oficiala Angular
- Foloseste CLAUDE.md/system prompts cu conventii actualizate

### Security vulnerabilities in generated code

**Studii arata ca AI-generated code poate fi mai vulnerabil decat human-written code** deoarece AI-ul optimizeaza pentru "arata corect" nu "este sigur".

**Vulnerabilitati comune:**
```typescript
// 1. Template injection
template: `<div [innerHTML]="${userContent}"></div>` // XSS

// 2. Insecure direct object reference
getUser(req) {
  return this.userRepo.findById(req.params.id); // Fara auth check
}

// 3. Sensitive data in logs
console.log('Login attempt:', { email, password }); // Parola in logs!

// 4. Insecure randomness
const token = Math.random().toString(36); // Predictibil!

// 5. Missing CSRF protection
// AI genereaza HTTP calls fara sa mentioneze CSRF tokens
```

### Over-reliance si skill atrophy

**Problema:** Cu cat folosesti mai mult AI, cu atat scrii mai putin cod manual, si skill-urile fundamentale se atrofiaza.

**Simptome:**
- Nu mai poti scrie o functie simpla fara AI
- Nu intelegi codul din proiectul propriu (l-a generat AI-ul)
- Debugging devine imposibil cand AI-ul nu stie raspunsul
- Interviurile fara AI devin foarte dificile

**Solutie:**
- Dedica timp sa scrii cod manual (coding exercises, side projects)
- Intelege fiecare linie de cod generat inainte de commit
- Foloseste AI ca accelerator, nu ca inlocuitor de gandire
- Mentine cunostinte fundamentale: data structures, algorithms, design patterns
- Periodic, lucreaza fara AI pentru a-ti verifica skill-urile

### License si IP concerns

**Problema fundamentala:** AI-ul a fost antrenat pe cod open-source cu diverse licente. Codul generat poate fi substantial similar cu cod sub licente restrictive (GPL, AGPL).

**Riscuri:**
- Cod generat care e copie aproape exacta a unui proiect GPL
- Potential de patent infringement
- Compania ta poate fi considerata ca foloseste cod nelicensiat
- GitHub Copilot a fost dat in judecata exact pe acest motiv

**Mitigare:**
- Foloseste tools cu indemnity clauses (Copilot Business, Claude for Enterprise)
- Nu accepta blindly blocuri mari de cod generat
- Ruleaza license scanning tools pe codebase
- Stabileste politica de companie pentru AI-generated code
- Documenteaza ce a fost generat de AI (pentru audit trail)

### Context window limitations

**Ce e context window:**
- Cantitatea de text (tokens) pe care AI-ul o poate "vedea" simultan
- GPT-4: ~128K tokens, Claude: ~200K tokens
- Pare mult, dar un proiect mediu Angular are milioane de tokens

**Impact practic:**
- AI-ul nu poate vedea tot proiectul simultan
- Poate genera cod inconsistent cu alte parti ale codebase-ului
- Pierde context in conversatii lungi
- Poate "uita" instructiuni de la inceputul conversatiei

**Mitigare:**
- Ofera context relevant, nu tot proiectul
- Foloseste @-references pentru fisiere specifice
- Repeta constrangerile importante in fiecare prompt
- Imparte task-urile mari in sub-task-uri cu context focusat

### Confidentialitate si cod proprietar

**Riscul:** Cand trimiti cod la un AI cloud service, acel cod trece prin servere externe.

**Intrebari critice:**
- Datele sunt folosite pentru training? (depinde de provider si plan)
- Codul ramane pe serverele provider-ului? Cat timp?
- Compliance cu GDPR, HIPAA, SOC2?
- NDA-ul cu clientul permite trimiterea codului la third parties?

**Solutii:**
- Enterprise plans (Copilot Business, Claude for Enterprise) - nu folosesc datele pentru training
- Self-hosted models (Ollama, vLLM) - totul ramane local
- Air-gapped environments pentru cod ultra-sensibil
- Politici clare de companie despre ce cod poate fi trimis la AI

---

## 10.7 Cum AI schimba rolul de Principal Engineer

### De la scrierea codului la review si directing

**Inainte:** Principal Engineer = cel mai bun coder din echipa, scrie codul cel mai complex.

**Acum:** Principal Engineer = cel care stie cel mai bine CE trebuie construit si VERIFICA ca e construit corect. Scrierea efectiva poate fi delegata (partial) AI-ului.

**Noile prioritati:**
1. **Definirea corecta a problemei** (cel mai important, AI nu poate face asta)
2. **Arhitectura si design** (trade-offs pe care AI nu le intelege)
3. **Review code** (uman sau AI-generated)
4. **Quality assurance** (standardele nu scad)
5. **Writing code** (inca important, dar proportia scade)

### Architecture si design devin si mai importante

AI-ul poate genera implementari, dar nu poate lua decizii arhitecturale bune deoarece:
- Nu intelege business context-ul complet
- Nu cunoaste constrangerile organizationale (echipe, skill-uri, timeline)
- Nu poate evalua trade-offs pe termen lung
- Nu stie ce a functionat si ce nu in proiectele anterioare

**Exemplu concret:**
```
Intrebare: "Cum implementam real-time updates?"

AI raspunde cu implementarea tehnica perfecta:
- WebSocket cu reconnect logic
- RxJS integration
- NgRx actions pentru sync

Dar AI nu stie:
- Infrastructura curenta nu suporta WebSocket-uri (load balancer vechi)
- Echipa nu are experienta cu WebSocket-uri
- Clientul nu are buget pentru infrastructure upgrade
- SSE (Server-Sent Events) ar fi suficient pentru use case-ul real

Principal Engineer-ul stie aceste lucruri si alege SSE.
```

### Viteza de prototyping creste dramatic

**Impact masurabil:**
- Un proof of concept care lua 2 saptamani acum ia 2 zile
- Spike-urile tehnice sunt mult mai rapide
- Poti explora 5 abordari in timpul in care explorai 1
- Demo-urile pentru stakeholders sunt disponibile mult mai repede

**Implicatii pentru Principal Engineer:**
- Mai multe prototipuri inseamna mai multe decizii de luat
- "Move fast" nu mai e o scuza - poti evalua mai multe optiuni
- Expectatiile stakeholderilor cresc ("daca AI-ul e asa rapid, de ce nu e gata?")
- Trebuie sa comunici ca prototip != productie

### Code quality bar must remain high

**Pericolul:** "AI-ul l-a generat rapid, da-l in productie!"

**Realitatea:** Viteza de generare NU inseamna calitate. Code review-ul devine MULT mai important cand volumul de cod generat creste.

**Principal Engineer-ul seteaza standardul:**
- Defineste coding standards si automated checks
- Configureaza linters, formatters, type checking strict
- Seteaza code coverage minime
- Revizuieste PR-uri cu atentie sporita la AI-generated code
- Refuza codul care nu e inteles de autor ("daca nu poti explica, nu merge in main")

### Mentoring echipei pe utilizarea eficienta a AI

**Noul skill set pe care Principal Engineer-ul trebuie sa il predea:**
- Cum sa scrii prompts eficiente
- Cum sa evaluezi output-ul AI-ului critic
- Cand sa folosesti AI si cand sa scrii manual
- Cum sa mentii skill-urile fundamentale
- Cum sa documentezi si sa urmaresti AI-generated code

**Anti-patterns de evitat in echipa:**
- Copy-paste din AI fara intelegere
- "AI-ul a zis ca e bine" ca justificare in code review
- Folosirea AI-ului ca scuza sa nu inveti
- Generare de cod doar ca sa para productiv

### Setarea de AI usage policies

Ca Principal Engineer, esti responsabil pentru:

1. **Ce tools sunt aprobate** si in ce configuratie (enterprise vs personal)
2. **Ce cod poate fi trimis la AI** (confidentiality boundaries)
3. **Standards pentru AI-generated code** (trebuie reviewuit? marcat? testat diferit?)
4. **Training budget** pentru AI tools
5. **Metrics** pentru a masura impactul AI-ului pe productivitate si calitate
6. **Escalation path** cand AI introduce probleme

### AI amplifica senior engineers mai mult decat juniors

**De ce:**
- Seniorii stiu CE sa ceara (prompt quality depinde de cunostinte)
- Seniorii pot evalua output-ul (stiu cand AI-ul greseste)
- Seniorii pot integra output-ul in arhitectura existenta
- Seniorii au mental models despre cum ar trebui sa arate codul bun

**Paradoxul junior-ului cu AI:**
- Juniorii pot genera cod rapid cu AI
- Dar nu pot evalua calitatea
- Pot introduce buguri subtile fara sa stie
- Pot "parea" productivi fara sa invete fundamentele
- Necesita MORE mentoring, nu less, intr-un mediu AI-enabled

**Implicatie:** Gapul intre senior si junior se poate mari, nu micsora. AI-ul nu egalizeaza - amplifica diferentele existente.

### "AI-native" development workflow

Un workflow modern integrat cu AI:

```
1. TASK DEFINITION (Human)
   - Business requirements
   - Acceptance criteria
   - Architecture decisions

2. PLANNING (Human + AI)
   - AI ajuta la task breakdown
   - AI sugereaza approach-uri
   - Human decide strategia

3. IMPLEMENTATION (AI + Human)
   - AI genereaza prima versiune
   - AI scrie teste
   - Human reviewieste si itereaza

4. REVIEW (Human + AI)
   - AI face prima trecere (linting, patterns)
   - Human face review-ul de logica si arhitectura
   - AI ajuta la addressing review comments

5. TESTING (AI + Human)
   - AI genereaza teste unitare si de integrare
   - Human defineste test scenarios
   - AI ajuta la debugging failures

6. DEPLOYMENT (Automated + Human)
   - CI/CD standard
   - Human approval gates
   - AI monitoring suggestions
```

---

## 10.8 Best practices pentru integrarea AI in workflow

### Start with low-risk tasks

**Tier 1 - Low risk (incepeti aici):**
- Generare de teste unitare pentru cod existent
- Documentatie si comments
- Boilerplate code (CRUD, forms, components)
- Regex generation
- Git commit messages
- Code formatting si cleanup

**Tier 2 - Medium risk:**
- Feature implementation cu review atent
- Refactoring cu test coverage buna
- Migration scripts (cu rollback plan)
- Performance optimization suggestions

**Tier 3 - High risk (atentie maxima):**
- Security-critical code (auth, encryption, validation)
- Financial calculations
- Data migration in productie
- Infrastructure as code
- Cod care interactioneaza cu sisteme externe fara sandbox

### Stabileste guidelines de echipa

```markdown
# Team AI Usage Guidelines (exemplu)

## Approved Tools
- GitHub Copilot Business (company license)
- Claude Code (with enterprise plan)
- ChatGPT Plus (for research only, no code paste)

## Rules
1. ALL AI-generated code must pass standard code review
2. Author is responsible for understanding every line
3. No proprietary business logic sent to free-tier AI services
4. Test coverage must be >= 80% for AI-generated features
5. AI-generated commits should be tagged: "AI-assisted"
6. Security-sensitive code requires manual implementation + review

## What NOT to send to AI
- Database credentials, API keys, secrets
- Customer PII (personal identifiable information)
- Proprietary algorithms (competitive advantage code)
- Code under NDA with specific clients
```

### Use AI for exploration, verify before production

```
Workflow recomandat:

1. EXPLORE - Foloseste AI sa genereze 2-3 abordari diferite
   "Show me 3 ways to implement infinite scroll in Angular"

2. EVALUATE - Analizeaza trade-offs
   "Compare these approaches for: performance, accessibility,
    bundle size, maintainability"

3. SELECT - Alege abordarea potrivita (decizie umana)

4. IMPLEMENT - Lasa AI-ul sa genereze implementarea aleasa

5. VERIFY - Review, test, profile
   - Code review manual
   - Ruleaza teste
   - Profile performance
   - Check accessibility

6. ITERATE - Cere AI-ului sa corecteze problemele gasite
```

### Combina multiple AI tools

**Workflow multi-tool:**
- **Claude Code** pentru implementare agentica (creaza fisiere, ruleaza teste)
- **Cursor** pentru editing interactiv si refactoring rapid
- **Copilot** pentru inline completions in timp ce scrii
- **ChatGPT/Claude web** pentru research si explicatii conceptuale

**Fiecare tool are strengths diferite:**
- Copilot: cel mai bun la inline completions (flow natural)
- Cursor: cel mai bun la multi-file editing (refactoring)
- Claude Code: cel mai bun la task-uri complexe autonome (features noi)
- ChatGPT: cel mai bun la explicatii si brainstorming

### Documenteaza AI-generated code in commits

```bash
# Optiunea 1: Tag in commit message
git commit -m "feat(users): add user profile component

AI-assisted: component structure and tests generated with Claude Code
Manual: business logic validation, error handling, accessibility"

# Optiunea 2: Co-authored-by
git commit -m "feat(users): add user profile component

Co-Authored-By: Claude Code <noreply@anthropic.com>"

# Optiunea 3: Git trailers
git commit -m "feat(users): add user profile component" \
  --trailer "AI-Tool: Claude Code" \
  --trailer "AI-Ratio: ~60%"
```

**De ce e important:**
- Audit trail pentru compliance
- Stii ce cod sa reviewuiesti cu atentie extra
- Metrics despre cat de mult AI foloseste echipa
- Debugging: "acest cod a fost generat, poate are pattern-uri unusual"

### Review periodic al eficacitatii AI tools

**Metrics de urmarit:**
- Timp de dezvoltare per feature (inainte vs dupa AI)
- Numar de buguri in AI-generated code vs manual code
- Code review time (creste sau scade?)
- Developer satisfaction cu tools
- Cost per developer per luna (subscriptions)
- Context switches intre tools

**Quarterly review:**
- Ce tools functioneaza bine?
- Ce tools nu isi justifica costul?
- Ce patterns noi au aparut?
- Ce probleme au cauzat AI tools?
- Trebuie actualizate guidelines?

### Human judgment ramane arbitrul final

**AI este un tool, nu un decision maker.**

```
CORECT:
Developer: "AI-ul sugereaza sa folosim IndexedDB pentru cache"
Principal: "Buna idee ca approach, dar avem deja un service worker
cache. Sa integram cu el in loc sa adaugam alt layer."

GRESIT:
Developer: "AI-ul a zis sa facem asa"
Principal: "OK, daca AI-ul a zis..."
```

**Principiul fundamental:** AI-ul propune, omul dispune. Responsabilitatea pentru cod ramane la developer si la echipa, nu la tool-ul AI.

---

## Intrebari frecvente de interviu

### 1. Cum folosesti AI tools in workflow-ul tau zilnic de dezvoltare? Da exemple concrete.

**Raspuns structurat:**
- Dimineata: review PR-uri cu AI assistance (identific probleme mai rapid)
- Development: Copilot pentru inline completions, Claude Code pentru features complexe
- Testing: AI genereaza testele initiale, eu le rafinez si adaug edge cases
- Debugging: descriu eroarea + context, AI sugereaza cauze si solutii
- Documentatie: AI genereaza prima versiune, eu o editez si completez

**Exemplu concret:** "Saptamana trecuta am migrat 15 componente de la NgModules la standalone. Am folosit Claude Code sa faca migratia automata, apoi am reviewuit fiecare componenta manual. Ce ar fi luat 3 zile a luat 4 ore, cu aceeasi calitate."

---

### 2. Cum verifici si evaluezi codul generat de AI inainte de a-l integra in proiect?

**Raspuns:**
- **Build check:** `ng build --configuration production` - verific ca se compileaza
- **Type safety:** TypeScript strict mode prinde multe probleme
- **Teste:** Rulez suita existenta + testele noi generate
- **Manual review:** Citesc fiecare linie si ma asigur ca inteleg
- **Security scan:** Verific input validation, sanitization, auth checks
- **Performance:** Caut memory leaks (unsubscribe), N+1 queries, change detection issues
- **Pattern consistency:** Compara cu conventiile proiectului
- **API verification:** Verific in documentatie ca API-urile folosite exista si sunt corecte

---

### 3. Care sunt principalele riscuri ale utilizarii AI in dezvoltarea software si cum le mitigati?

**Raspuns:** Identific 5 categorii principale de riscuri:

1. **Hallucinations** - AI genereaza API-uri inexistente sau logica gresita
   - Mitigare: review manual, teste automate, type checking strict

2. **Security vulnerabilities** - cod generat cu XSS, injection, exposure
   - Mitigare: security scanning, code review focusat pe securitate, OWASP checklist

3. **Skill atrophy** - developerii devin dependenti de AI
   - Mitigare: coding exercises fara AI, pair programming, mentoring

4. **Confidentiality** - cod proprietar trimis la servere externe
   - Mitigare: enterprise plans, self-hosted models, politici clare

5. **IP/License** - cod generat similar cu proiecte open-source restrictive
   - Mitigare: license scanning, indemnity clauses, review de blocuri mari

---

### 4. Cum ar trebui sa se schimbe procesul de code review cand echipa foloseste AI tools?

**Raspuns:**
- **Volumul creste** - AI genereaza cod mai rapid, deci mai multe PR-uri
- **Focus-ul se schimba** - mai putin pe sintaxa, mai mult pe logica si arhitectura
- **Scepticism crescut** - codul AI arata "prea bine" uneori, trebuie verificat mai atent
- **Checklist extins** - adaugi verificari specifice AI: API-uri reale? Pattern-uri actuale? Edge cases?
- **Autorship verification** - "Poti explica de ce ai ales aceasta abordare?"
- **Automated gates** - linters, type checking, security scanning devin obligatorii
- **Two-pass review** - prima trecere automata (AI-assisted), a doua trecere manuala focusata pe business logic

---

### 5. Cum ai implementa o politica de utilizare AI intr-o echipa de dezvoltare? Ce ar include?

**Raspuns:** Politica ar acoperi:

1. **Tools aprobate** - lista specifica cu versiuni enterprise
2. **Clasificarea datelor** - ce cod poate fi trimis la AI (public, intern, confidential)
3. **Quality gates** - AI-generated code trece prin acelasi review ca orice cod
4. **Responsibility** - autorul ramane responsabil, nu AI-ul
5. **Documentation** - cum marcam codul generat de AI (commit tags)
6. **Training** - onboarding pe AI tools pentru toti membrii echipei
7. **Metrics** - cum masuram impactul (pozitiv si negativ)
8. **Review cadence** - quarterly review al politicii si tool-urilor
9. **Escalation** - ce facem cand AI introduce un bug in productie
10. **Compliance** - aliniere cu GDPR, NDA-uri, regulatory requirements

---

### 6. In ce situatii ai alege sa NU folosesti AI pentru generarea de cod?

**Raspuns:**

- **Security-critical code** - authentication, encryption, authorization logic
  - Riscul e prea mare, un bug subtil poate fi catastrofal
- **Core business logic** - algoritmii care definesc valoarea produsului
  - AI nu intelege domeniul; de exemplu, calculul unui scor de risc financiar necesita expertiza umana
- **Code sub NDA strict** - cand contractul interzice explicit trimiterea codului la third parties
- **Cand nu inteleg domeniul** - daca nu pot evalua corectitudinea output-ului
  - Exemplu: un algoritm medical unde nu am expertiza sa verific
- **Infrastructure critice** - Terraform/Pulumi pentru productie
  - O greseala poate sterge baze de date sau expune sisteme
- **Cand invat ceva nou** - vreau sa inteleg profund, nu sa primesc raspunsul
  - Folosesc AI pentru EXPLICATII, nu pentru GENERARE in acest caz

---

### 7. Cum crezi ca AI va schimba rolul de senior/principal engineer in urmatorii 2-3 ani?

**Raspuns:**

**Shift-uri majore:**
- De la "cel mai bun coder" la "cel mai bun reviewer si architect"
- De la "scriu cod 70% din timp" la "dirijez AI si reviewuiesc 70% din timp"
- Prototyping-ul devine aproape instant - focus pe evaluare si decizie
- Mentoring-ul se extinde la "cum sa folosesti AI eficient"

**Ce devine MAI important:**
- System design si architectural thinking
- Problem decomposition
- Code review skills
- Communication si stakeholder management
- Security awareness
- Domain expertise

**Ce devine MAI PUTIN important:**
- Memorarea de API-uri si syntax
- Scrierea de boilerplate
- Viteza de typing
- Cunoasterea a 15 limbaje de programare

**Predictie:** Rolul de Principal Engineer devine mai strategic si mai putin tactical. AI-ul preia executia repetitiva; expertiza umana se concentreaza pe decizii, trade-offs si calitate.

---

### 8. Descrie un scenariu in care AI-ul ti-a generat cod gresit si cum ai descoperit problema.

**Raspuns exemplu:**

"Foloseam Claude sa genereze un HTTP interceptor pentru refresh token logic in Angular. Codul arata perfect - intercepta 401, facea refresh, retrimitea request-ul original.

Problema: in cazul in care 3 request-uri primeau 401 simultan, interceptorul facea 3 refresh-uri paralele. Race condition clasic - doar primul refresh reusea, celelalte doua primeau 'invalid refresh token' si delogau userul.

Cum am descoperit: am scris un test E2E care simula network latency. Testul a falat intermitent (flaky test). Am investigat si am descoperit race condition-ul.

Fix: am adaugat un `shareReplay` / queueing mechanism care face un singur refresh si queue-uieste celelalte request-uri. Am cerut AI-ului sa genereze fix-ul, dar i-am descris exact problema - 'add request queueing during token refresh to prevent multiple simultaneous refresh calls'.

Lectie: AI-ul genereaza happy path-ul perfect. Edge cases si concurrency issues necesita gandire umana si teste robuste."

---

### 9. Cum ai folosi AI intr-un interviu tehnic (daca ti se permite)? Ce strategii ai aplica?

**Raspuns:**

**Strategie:**
1. **Nu incepe cu AI** - citeste problema, gandeste-te 2-3 minute, formeaza un plan mental
2. **Descompune problema** - scrie sub-probleme pe care le vei rezolva (arata interviewer-ului gandirea)
3. **Foloseste AI pentru implementare, nu pentru gandire**
   - "Implement a debounced search with these requirements: ..."
   - NU "Solve this interview problem for me"
4. **Verifica output-ul vocal** - "AI-ul a generat asta, dar vad ca lipseste error handling pentru cazul cand..."
5. **Itereaza** - cere imbunatatiri specifice, nu regenerare completa
6. **Explica deciziile** - "Am ales sa folosesc signals in loc de BehaviorSubject pentru ca..."

**Ce impresionanaza interviewer-ul:**
- Stii cand AI-ul greseste si corectezi
- Folosesti AI ca accelerator, nu ca stampila
- Esti transparent despre ce e AI-generated si ce e gandirea ta
- Poti explica fiecare decizie tehnica

**Ce NU impresionanaza:**
- Copy-paste fara intelegere
- "Nu stiu, dar AI-ul a zis asa"
- Acceptarea orba a primului output
- Incercarea de a ascunde utilizarea AI-ului

---

### 10. Ce inseamna "prompt engineering" in contextul dezvoltarii software si de ce e important?

**Raspuns:**

Prompt engineering in contextul dezvoltarii software inseamna abilitatea de a comunica eficient cu un AI tool pentru a obtine cod de calitate. Este o competenta tehnica noua, la fel de importanta ca si skill-ul de a scrie query-uri SQL bune sau de a configura un build system.

**Principii cheie:**
- **Specificitate** - limbaj, framework, versiune, constrangeri
- **Context** - cod existent, arhitectura, conventii de proiect
- **Iteratie** - rafineaza in pasi, nu cere totul dintr-o data
- **Exemple** - arata pattern-ul dorit (few-shot prompting)
- **Constrangeri** - spune ce NU vrei, nu doar ce vrei
- **Verificare** - cere AI-ului sa explice rationamentul

**De ce e important pentru un Principal Engineer:**
- Diferenta intre un prompt bun si unul slab poate fi 10x in calitatea output-ului
- Economiseste ore de iteratie si corectie
- Seteaza standardul pentru echipa
- Devine un skill de baza in interviuri (cum era Git acum 10 ani)
- Un prompt bine scris este, in esenta, o specificatie clara - skill fundamental de engineering

**Meta-observatie:** Prompt engineering bun necesita aceleasi skill-uri ca si comunicarea buna in general: claritate, specificitate, context, si capacitatea de a anticipa ambiguitatile. Engineerii care comunica bine cu oamenii comunica bine si cu AI-ul.
