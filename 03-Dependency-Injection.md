# 03. Dependency Injection in Angular

## Cuprins

- [Ierarhia de injectori](#ierarhia-de-injectori)
- [InjectionToken si provideri custom](#injectiontoken-si-provideri-custom)
- [forwardRef()](#forwardref)
- [Resolution modifiers](#resolution-modifiers)
- [providedIn options](#providedin-options)
- [Multi-providers](#multi-providers)
- [Tree-shakable providers](#tree-shakable-providers)
- [inject() function vs constructor injection](#inject-function-vs-constructor-injection)
- [Intrebari frecvente de interviu](#intrebari-frecvente-de-interviu)

---

## Ierarhia de injectori

Angular foloseste un sistem ierarhic de injectori care determina cum sunt rezolvate dependentele. Intelegerea acestei ierarhii este esentiala pentru un Principal Engineer, deoarece afecteaza direct scopul serviciilor, performanta si arhitectura aplicatiei.

### Diagrama completa a rezolutiei

```
Cerere de dependenta
        |
        v
  Element Injector (component/directive @Component.providers)
        |  (nu gaseste)
        v
  Parent Element Injector (component parinte)
        |  (nu gaseste, continua in sus prin DOM)
        v
  ...pana la Root Component Element Injector
        |  (nu gaseste)
        v
  Module Injector (NgModule providers al componentei)
        |  (nu gaseste)
        v
  Parent Module Injector (module parinte, inclusiv lazy-loaded)
        |  (nu gaseste)
        v
  Root Injector (providedIn: 'root' sau AppModule providers)
        |  (nu gaseste)
        v
  Platform Injector (servicii partajate intre aplicatii)
        |  (nu gaseste)
        v
  NullInjector (arunca eroare sau returneaza null cu @Optional)
```

### Platform Injector

Platform Injector este cel mai de sus nivel in ierarhie. Este creat la bootstrap-ul platformei si permite partajarea serviciilor intre mai multe aplicatii Angular care ruleaza pe aceeasi pagina (scenarii micro-frontend).

```typescript
// Cand ai mai multe aplicatii Angular pe aceeasi pagina
import { createPlatform, platformBrowserDynamic } from '@angular/platform-browser-dynamic';

const platform = platformBrowserDynamic([
  {
    provide: SharedAnalyticsService,
    useClass: SharedAnalyticsService
  }
]);

// Ambele aplicatii vor primi aceeasi instanta
platform.bootstrapModule(AppModule1);
platform.bootstrapModule(AppModule2);
```

In practica, Platform Injector este rar folosit direct. Devine relevant in scenarii de micro-frontend sau cand testezi cu `TestBed.initTestEnvironment()`.

### Root Injector

Root Injector este cel mai folosit nivel. Serviciile inregistrate aici sunt singleton-uri la nivel de aplicatie. Exista doua modalitati de a inregistra:

```typescript
// Metoda preferata - tree-shakable
@Injectable({
  providedIn: 'root'
})
export class AuthService {
  private currentUser = signal<User | null>(null);

  login(credentials: Credentials): Observable<User> {
    return this.http.post<User>('/api/login', credentials).pipe(
      tap(user => this.currentUser.set(user))
    );
  }
}

// Metoda veche - in AppModule (NU este tree-shakable)
@NgModule({
  providers: [AuthService]  // evita aceasta abordare pentru servicii noi
})
export class AppModule {}
```

### Module Injector

Fiecare NgModule care declara `providers` creaza propriul injector. Acest lucru devine important mai ales cu lazy-loaded modules, deoarece fiecare modul lazy primeste propria instanta a serviciilor.

```typescript
// Acest modul este lazy-loaded
@NgModule({
  providers: [
    OrderProcessingService  // instanta unica pentru acest modul lazy
  ]
})
export class OrdersModule {}

// OrderProcessingService va fi o instanta DIFERITA de cea din alt modul lazy
// Dar serviciile din Root Injector (providedIn: 'root') raman singleton
```

**Atentie la capcana clasica:** Daca un serviciu este declarat in `providers` al unui modul lazy-loaded, componentele din acel modul vor primi o instanta diferita fata de componentele din alte module. Aceasta nu este o problema daca serviciul este `providedIn: 'root'`, dar poate crea bug-uri subtile cu module providers.

```typescript
// Scenariul problemei
@NgModule({
  providers: [CartService]  // GRESEALA daca vrei un singur cos global
})
export class ShopModule {}  // lazy-loaded

@NgModule({
  providers: [CartService]  // alta instanta!
})
export class CheckoutModule {}  // lazy-loaded

// Solutia corecta:
@Injectable({ providedIn: 'root' })
export class CartService {}  // singleton global, indiferent de module
```

### Element Injector

Element Injector este creat pentru fiecare componenta si directiva. Este cel mai granular nivel si permite izolarea serviciilor la nivel de componenta si sub-arborele sau din DOM.

```typescript
@Component({
  selector: 'app-editor',
  providers: [
    EditorStateService  // instanta noua pentru FIECARE <app-editor>
  ],
  template: `
    <app-toolbar />
    <app-canvas />
    <app-properties-panel />
  `
})
export class EditorComponent {
  constructor(private editorState: EditorStateService) {}
}

// Componentele copil primesc ACEEASI instanta prin mostenire
@Component({
  selector: 'app-toolbar',
  template: `...`
})
export class ToolbarComponent {
  // Primeste instanta EditorStateService de la parinte (EditorComponent)
  constructor(private editorState: EditorStateService) {}
}
```

Un pattern avansat foloseste `viewProviders` pentru a limita vizibilitatea doar la template-ul componentei, nu si la content projection:

```typescript
@Component({
  selector: 'app-tabs',
  // providers - vizibil pentru copii din template SI content projection
  // viewProviders - vizibil DOAR pentru copii din template
  viewProviders: [TabGroupService],
  template: `
    <div class="tab-header">
      <!-- TabGroupService este disponibil aici -->
      <ng-content select="app-tab"></ng-content>
      <!-- TabGroupService NU este disponibil pentru <app-tab> proiectat -->
    </div>
  `
})
export class TabsComponent {}
```

### providers vs viewProviders - Diferenta critica

```typescript
@Component({
  selector: 'app-parent',
  providers: [ServiceA],         // vizibil prin Element Injector pentru toti copiii
  viewProviders: [ServiceB],     // vizibil DOAR pentru copiii din view (template)
  template: `
    <!-- ServiceB disponibil aici, in view -->
    <app-view-child></app-view-child>

    <!-- ServiceB NU este disponibil pentru continutul proiectat -->
    <ng-content></ng-content>
  `
})
export class ParentComponent {}

// Folosire:
// <app-parent>
//   <app-projected-child></app-projected-child> <!-- NU vede ServiceB -->
// </app-parent>
```

---

## InjectionToken si provideri custom

### Crearea unui InjectionToken

`InjectionToken` este mecanismul prin care injectam valori care nu sunt clase (configuratii, constante, obiecte) sau cand vrem sa injectam interfete (TypeScript nu pastreaza interfetele la runtime).

```typescript
import { InjectionToken } from '@angular/core';

// Token simplu cu tip generic
export interface AppConfig {
  apiBaseUrl: string;
  featureFlags: Record<string, boolean>;
  maxRetries: number;
  environment: 'development' | 'staging' | 'production';
}

export const APP_CONFIG = new InjectionToken<AppConfig>('app.config');

// Token cu valoare implicita (tree-shakable)
export const API_BASE_URL = new InjectionToken<string>('api.base.url', {
  providedIn: 'root',
  factory: () => 'https://api.example.com/v1'
});

// Token cu factory care foloseste alte dependente
export const AUTH_HEADER = new InjectionToken<string>('auth.header', {
  providedIn: 'root',
  factory: () => {
    const authService = inject(AuthService);
    return `Bearer ${authService.getToken()}`;
  }
});
```

### useClass

Inlocuieste o clasa cu alta implementare. Util pentru testing, strategii diferite per mediu, sau cand vrei sa inlocuiesti o implementare concreta:

```typescript
// Interfata abstracta (clasa abstracta, nu interfata TS)
export abstract class LoggerService {
  abstract log(message: string, context?: Record<string, unknown>): void;
  abstract error(message: string, error?: Error): void;
  abstract warn(message: string): void;
}

// Implementare pentru productie
export class CloudLoggerService extends LoggerService {
  private http = inject(HttpClient);

  log(message: string, context?: Record<string, unknown>): void {
    this.http.post('/api/logs', {
      level: 'info',
      message,
      context,
      timestamp: Date.now()
    }).subscribe();
  }

  error(message: string, error?: Error): void {
    this.http.post('/api/logs', {
      level: 'error',
      message,
      stack: error?.stack,
      timestamp: Date.now()
    }).subscribe();
  }

  warn(message: string): void {
    this.http.post('/api/logs', { level: 'warn', message, timestamp: Date.now() })
      .subscribe();
  }
}

// Implementare pentru development
export class ConsoleLoggerService extends LoggerService {
  log(message: string, context?: Record<string, unknown>): void {
    console.log(`[INFO] ${message}`, context ?? '');
  }

  error(message: string, error?: Error): void {
    console.error(`[ERROR] ${message}`, error);
  }

  warn(message: string): void {
    console.warn(`[WARN] ${message}`);
  }
}

// Configurare in functie de mediu
@NgModule({
  providers: [
    {
      provide: LoggerService,
      useClass: environment.production ? CloudLoggerService : ConsoleLoggerService
    }
  ]
})
export class CoreModule {}
```

### useValue

Injecteaza o valoare statica. Ideal pentru configuratii, constante, sau obiecte deja instantiate:

```typescript
// Configuratie statica
const appConfig: AppConfig = {
  apiBaseUrl: environment.apiUrl,
  featureFlags: {
    darkMode: true,
    betaFeatures: false,
    newCheckout: true
  },
  maxRetries: 3,
  environment: environment.production ? 'production' : 'development'
};

@NgModule({
  providers: [
    { provide: APP_CONFIG, useValue: appConfig },
    { provide: 'APP_VERSION', useValue: '2.5.0' }
  ]
})
export class AppModule {}

// Injectarea obiectelor browser (util si pentru SSR)
export const WINDOW = new InjectionToken<Window>('window', {
  providedIn: 'root',
  factory: () => {
    if (typeof window !== 'undefined') {
      return window;
    }
    // Pentru SSR, returneaza un mock minimal
    return {} as Window;
  }
});

export const LOCAL_STORAGE = new InjectionToken<Storage>('localStorage', {
  providedIn: 'root',
  factory: () => {
    const platformId = inject(PLATFORM_ID);
    if (isPlatformBrowser(platformId)) {
      return window.localStorage;
    }
    // Mock pentru SSR
    return {
      getItem: () => null,
      setItem: () => {},
      removeItem: () => {},
      clear: () => {},
      length: 0,
      key: () => null
    } as Storage;
  }
});

export const DOCUMENT_REF = new InjectionToken<Document>('document', {
  providedIn: 'root',
  factory: () => inject(DOCUMENT)  // din @angular/common
});
```

### useFactory

Cea mai flexibila optiune. Permite crearea conditionala, initializare complexa, si accesul la alte dependente:

```typescript
// Factory cu dependente injectate
export const API_CLIENT = new InjectionToken<ApiClient>('api.client');

@NgModule({
  providers: [
    {
      provide: API_CLIENT,
      useFactory: (
        http: HttpClient,
        config: AppConfig,
        auth: AuthService
      ): ApiClient => {
        const client = new ApiClient(http, config.apiBaseUrl);
        client.setDefaultHeaders({
          'Authorization': `Bearer ${auth.getToken()}`,
          'X-App-Version': config.environment
        });
        client.setRetryPolicy(config.maxRetries);
        return client;
      },
      deps: [HttpClient, APP_CONFIG, AuthService]
    }
  ]
})
export class CoreModule {}

// Factory conditionala bazata pe mediu
@NgModule({
  providers: [
    {
      provide: CacheService,
      useFactory: (platformId: object) => {
        if (isPlatformBrowser(platformId)) {
          return new IndexedDbCacheService();
        }
        return new InMemoryCacheService();
      },
      deps: [PLATFORM_ID]
    }
  ]
})
export class CacheModule {}

// Varianta moderna cu inject() in factory (fara deps)
export const FEATURE_FLAGS = new InjectionToken<FeatureFlags>('feature.flags', {
  providedIn: 'root',
  factory: () => {
    const config = inject(APP_CONFIG);
    const userService = inject(UserService);

    // Logica complexa de determinare a feature flags
    const baseFlags = config.featureFlags;
    const userFlags = userService.getCurrentUser()?.featureOverrides ?? {};

    return { ...baseFlags, ...userFlags };
  }
});
```

### useExisting

Creaza un alias pentru un provider existent. Util cand vrei ca mai multe token-uri sa pointeze catre aceeasi instanta:

```typescript
// Scenariul clasic: implementare unica, multiple interfete
@Injectable({ providedIn: 'root' })
export class AppStateService implements OnDestroy {
  private state = signal<AppState>(initialState);

  select<T>(selector: (state: AppState) => T): Signal<T> {
    return computed(() => selector(this.state()));
  }

  dispatch(action: Action): void {
    const newState = reducer(this.state(), action);
    this.state.set(newState);
  }

  ngOnDestroy(): void {
    // cleanup
  }
}

// Clasa abstracta pentru read-only access
export abstract class StateReader {
  abstract select<T>(selector: (state: AppState) => T): Signal<T>;
}

// Clasa abstracta pentru write-only access
export abstract class StateWriter {
  abstract dispatch(action: Action): void;
}

@NgModule({
  providers: [
    AppStateService,
    { provide: StateReader, useExisting: AppStateService },
    { provide: StateWriter, useExisting: AppStateService }
  ]
})
export class StateModule {}

// Componentele primesc doar ce au nevoie (Interface Segregation Principle)
@Component({ /* ... */ })
export class DashboardComponent {
  // Poate doar citi state-ul
  private stateReader = inject(StateReader);
  users = this.stateReader.select(s => s.users);
}

@Component({ /* ... */ })
export class AdminComponent {
  // Poate si scrie
  private stateWriter = inject(StateWriter);

  resetUsers(): void {
    this.stateWriter.dispatch({ type: 'RESET_USERS' });
  }
}
```

---

## forwardRef()

### Cand si de ce este necesar

`forwardRef()` rezolva problema referintelor circulare si a ordinii de declarare in TypeScript. JavaScript (la runtime) nu poate accesa o clasa inainte ca aceasta sa fie definita, dar uneori dependentele formeaza un cerc.

### Problema clasica: ordinea declararii

```typescript
// EROARE - UserService nu este inca definit la momentul evaluarii
@Injectable({ providedIn: 'root' })
export class AuthService {
  constructor(private userService: UserService) {}  // ReferenceError la runtime
}

@Injectable({ providedIn: 'root' })
export class UserService {
  constructor(private authService: AuthService) {}
}
```

### Solutia cu forwardRef()

```typescript
import { forwardRef, Inject, Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class AuthService {
  constructor(
    @Inject(forwardRef(() => UserService))
    private userService: UserService
  ) {}

  validateToken(token: string): boolean {
    const user = this.userService.getUserByToken(token);
    return !!user;
  }
}

@Injectable({ providedIn: 'root' })
export class UserService {
  constructor(private authService: AuthService) {}

  getUserByToken(token: string): User | null {
    // ...implementare
    return null;
  }
}
```

### Pattern mai comun: NG_VALIDATORS cu componenta care nu este inca definita

```typescript
import { Component, forwardRef } from '@angular/core';
import { NG_VALUE_ACCESSOR, ControlValueAccessor } from '@angular/forms';

@Component({
  selector: 'app-star-rating',
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      // Fara forwardRef, StarRatingComponent nu este inca definit
      // deoarece decoratorul @Component este evaluat INAINTE ca clasa sa fie procesata
      useExisting: forwardRef(() => StarRatingComponent),
      multi: true
    }
  ],
  template: `
    <span
      *ngFor="let star of stars; let i = index"
      (click)="rate(i + 1)"
      [class.filled]="i < value">
      &#9733;
    </span>
  `
})
export class StarRatingComponent implements ControlValueAccessor {
  stars = [1, 2, 3, 4, 5];
  value = 0;
  private onChange: (value: number) => void = () => {};
  private onTouched: () => void = () => {};

  rate(value: number): void {
    this.value = value;
    this.onChange(value);
    this.onTouched();
  }

  writeValue(value: number): void {
    this.value = value ?? 0;
  }

  registerOnChange(fn: (value: number) => void): void {
    this.onChange = fn;
  }

  registerOnTouched(fn: () => void): void {
    this.onTouched = fn;
  }
}
```

### Recomandarea pentru Principal Engineer

In codul modern Angular, `forwardRef()` este rar necesar daca:
- Structurezi corect modulele si eviti dependentele circulare
- Folosesti `inject()` in loc de constructor injection (rezolva unele cazuri)
- Separi responsabilitatile in servicii distincte

Totusi, `forwardRef()` ramane obligatoriu pentru pattern-ul `NG_VALUE_ACCESSOR` / `NG_VALIDATORS` cu `useExisting`, deoarece clasa componenta nu este inca disponibila in momentul evaluarii decoratorului.

---

## Resolution modifiers

Resolution modifiers controleaza CUM si UNDE Angular cauta dependentele in ierarhia de injectori. Sunt decoratori aplicati pe parametrii constructorului sau flag-uri in `inject()`.

### @Optional()

Returneaza `null` daca dependenta nu este gasita, in loc sa arunce eroare.

```typescript
// Cu constructor injection
@Component({ /* ... */ })
export class NotificationComponent {
  constructor(@Optional() private analytics: AnalyticsService | null) {}

  showNotification(message: string): void {
    this.displayToast(message);
    // Analytics poate fi null daca nu este configurat
    this.analytics?.trackEvent('notification_shown', { message });
  }
}

// Cu inject() - varianta moderna
@Component({ /* ... */ })
export class NotificationComponent {
  private analytics = inject(AnalyticsService, { optional: true });

  showNotification(message: string): void {
    this.displayToast(message);
    this.analytics?.trackEvent('notification_shown', { message });
  }
}

// Exemplu practic: componenta care functioneaza cu sau fara FormGroup parinte
@Component({
  selector: 'app-search-input',
  template: `
    <input
      [formControl]="searchControl"
      (input)="onSearch($event)"
      placeholder="Cauta..." />
  `
})
export class SearchInputComponent implements OnInit {
  // Poate fi folosit standalone SAU in interiorul unui <form>
  private parentForm = inject(ControlContainer, { optional: true });
  searchControl = new FormControl('');

  ngOnInit(): void {
    if (this.parentForm) {
      // Integreaza cu formularul parinte
      console.log('Integrat in form:', this.parentForm.path);
    }
  }
}
```

### @Self()

Restrictioneaza cautarea DOAR la injectorul curent (Element Injector al componentei). Nu urca in ierarhie.

```typescript
@Component({
  selector: 'app-user-profile',
  providers: [UserCacheService],  // Instanta locala OBLIGATORIE
  template: `<div>{{ user().name }}</div>`
})
export class UserProfileComponent {
  // @Self() garanteaza ca foloseste instanta LOCALA, nu una mostenita
  private cache = inject(UserCacheService, { self: true });
  user = this.cache.getCurrentUser();
}

// Daca UserCacheService NU este in providers-ul acestei componente
// Angular va arunca eroare, chiar daca un parinte il ofera

// Pattern practic: validare ca o directiva este folosita pe elementul corect
@Directive({
  selector: '[appTooltip]'
})
export class TooltipDirective implements OnInit {
  // Verifica ca directiva este aplicata pe un element care are NgModel
  private ngModel = inject(NgModel, { self: true, optional: true });

  ngOnInit(): void {
    if (!this.ngModel) {
      console.warn('appTooltip functioneaza optim pe elemente cu NgModel');
    }
  }
}
```

### @SkipSelf()

Sare peste injectorul curent si cauta DOAR in injectori parinti. Esential pentru pattern-ul de servicii ierarhice.

```typescript
// Pattern clasic: tree structure cu servicii ierarhice
@Injectable()
export class TreeNodeService {
  children: TreeNodeService[] = [];
  depth: number;

  constructor(
    @Optional() @SkipSelf() private parent: TreeNodeService | null
  ) {
    this.depth = parent ? parent.depth + 1 : 0;
    parent?.registerChild(this);
  }

  registerChild(child: TreeNodeService): void {
    this.children.push(child);
  }

  getPath(): string {
    if (!this.parent) return 'root';
    return `${this.parent.getPath()} > node-${this.depth}`;
  }
}

@Component({
  selector: 'app-tree-node',
  providers: [TreeNodeService],  // Fiecare nod primeste propria instanta
  template: `
    <div [style.padding-left.px]="nodeService.depth * 20">
      Nod la adancimea {{ nodeService.depth }}
      <p>Cale: {{ nodeService.getPath() }}</p>
      <ng-content></ng-content>
    </div>
  `
})
export class TreeNodeComponent {
  constructor(public nodeService: TreeNodeService) {}
}

// Varianta cu inject()
@Component({
  selector: 'app-tree-node',
  providers: [TreeNodeService],
  template: `...`
})
export class TreeNodeComponent {
  nodeService = inject(TreeNodeService);
  // SkipSelf se aplica IN serviciu, nu in componenta
}

// Alt exemplu: override local cu acces la implementarea parinte
@Injectable()
export class ThemeService {
  private parentTheme = inject(ThemeService, { skipSelf: true, optional: true });
  private localOverrides = signal<Partial<Theme>>({});

  theme = computed<Theme>(() => {
    const base = this.parentTheme?.theme() ?? DEFAULT_THEME;
    return { ...base, ...this.localOverrides() };
  });

  override(overrides: Partial<Theme>): void {
    this.localOverrides.set(overrides);
  }
}

@Component({
  selector: 'app-sidebar',
  providers: [ThemeService],  // Override local de tema
  template: `
    <div [style.background]="themeService.theme().background">
      <!-- Sub-arborele va folosi tema override -->
      <ng-content></ng-content>
    </div>
  `
})
export class SidebarComponent {
  themeService = inject(ThemeService);

  constructor() {
    this.themeService.override({ background: '#1a1a2e', text: '#ffffff' });
  }
}
```

### @Host()

Opreste cautarea la componenta gazda (host). Nu urca mai sus de componenta care gazduieste directiva sau content projection.

```typescript
// Definitia "host" depinde de context:
// - Pentru o directiva aplicata pe o componenta: componenta respectiva este host-ul
// - Pentru content projection: componenta care face <ng-content> este host-ul

@Directive({
  selector: '[appHighlight]'
})
export class HighlightDirective {
  // Cauta FormGroupDirective DOAR pana la componenta host
  private formGroup = inject(FormGroupDirective, { host: true, optional: true });

  constructor(private el: ElementRef) {
    if (this.formGroup) {
      // Avem acces la formularul host
      console.log('Directiva aplicata in contextul unui FormGroup');
    }
  }
}

// Exemplu practic: tab system
@Component({
  selector: 'app-tab-group',
  providers: [TabGroupService],
  template: `
    <div class="tabs-header">
      <button *ngFor="let tab of tabGroup.tabs()"
              (click)="tabGroup.selectTab(tab)"
              [class.active]="tabGroup.isActive(tab)">
        {{ tab.label }}
      </button>
    </div>
    <div class="tab-content">
      <ng-content></ng-content>
    </div>
  `
})
export class TabGroupComponent {
  tabGroup = inject(TabGroupService);
}

@Component({
  selector: 'app-tab',
  template: `
    <div [hidden]="!isActive()">
      <ng-content></ng-content>
    </div>
  `
})
export class TabComponent implements OnInit {
  @Input() label = '';

  // @Host() - cauta TabGroupService DOAR pana la host component (TabGroupComponent)
  // Nu va gasi TabGroupService dintr-un TabGroup parinte diferit
  private tabGroup = inject(TabGroupService, { host: true });

  isActive = computed(() => this.tabGroup.isActive(this));

  ngOnInit(): void {
    this.tabGroup.registerTab(this);
  }
}
```

### Combinarea modifierilor

Modifierii pot fi combinati pentru un control fin al rezolutiei:

```typescript
@Component({
  selector: 'app-widget',
  providers: [WidgetStateService],
  template: `...`
})
export class WidgetComponent {
  // Cauta DOAR la parinti (nu local), si returneaza null daca nu gaseste
  private parentWidget = inject(WidgetStateService, {
    skipSelf: true,
    optional: true
  });

  // Cauta DOAR local, si returneaza null daca nu gaseste
  private localConfig = inject(WidgetConfig, {
    self: true,
    optional: true
  });
}

// Tabel rezumativ al combinatiilor utile:
//
// | Combinatie               | Comportament                                           |
// |--------------------------|--------------------------------------------------------|
// | @Optional()              | null daca nu gaseste (in loc de eroare)                 |
// | @Self()                  | doar injectorul curent                                 |
// | @SkipSelf()              | sare peste curent, cauta in parinti                    |
// | @Host()                  | pana la host component                                 |
// | @Self() + @Optional()    | local sau null                                         |
// | @SkipSelf() + @Optional()| parinti sau null (pattern ierarhic clasic)              |
// | @Host() + @Optional()    | pana la host sau null                                  |
```

---

## providedIn options

### providedIn: 'root'

Optiunea implicita si cea mai folosita. Creaza un singleton la nivel de aplicatie, care este tree-shakable.

```typescript
@Injectable({
  providedIn: 'root'
})
export class UserPreferencesService {
  private preferences = signal<UserPreferences>(DEFAULT_PREFERENCES);

  // O singura instanta in toata aplicatia
  // Daca nicio componenta/serviciu nu il injecteaza, este eliminat din bundle
  getPreference<K extends keyof UserPreferences>(key: K): Signal<UserPreferences[K]> {
    return computed(() => this.preferences()[key]);
  }

  updatePreference<K extends keyof UserPreferences>(
    key: K,
    value: UserPreferences[K]
  ): void {
    this.preferences.update(prefs => ({ ...prefs, [key]: value }));
  }
}
```

**Cand folosesti `providedIn: 'root'`:**
- Servicii care trebuie sa fie singleton (AuthService, HttpClient wrapper, State management)
- Servicii utilitare (DateFormatter, ValidationHelpers)
- Orice serviciu care nu are nevoie de instante multiple

### providedIn: 'any'

Creaza o instanta separata pentru fiecare lazy-loaded module. Modulele eager-loaded impart aceeasi instanta ca root.

```typescript
@Injectable({
  providedIn: 'any'
})
export class ModuleScopedCacheService {
  private cache = new Map<string, unknown>();

  // Fiecare modul lazy are propriul cache independent
  get<T>(key: string): T | undefined {
    return this.cache.get(key) as T | undefined;
  }

  set(key: string, value: unknown): void {
    this.cache.set(key, value);
  }

  clear(): void {
    this.cache.clear();
  }
}
```

**Cand folosesti `providedIn: 'any'`:**
- Cache-uri specifice modulului
- State local per feature module lazy-loaded
- Cand vrei izolare intre sectiuni ale aplicatiei

**Nota:** `providedIn: 'any'` este relativ rar folosit si poate crea confuzie. De cele mai multe ori, `providedIn: 'root'` sau component-level providers sunt suficiente.

### providedIn: 'platform'

Partajeaza instanta intre toate aplicatiile Angular de pe aceeasi pagina.

```typescript
@Injectable({
  providedIn: 'platform'
})
export class SharedTelemetryService {
  private events: TelemetryEvent[] = [];

  // Aceeasi instanta pentru toate aplicatiile Angular de pe pagina
  track(appId: string, event: TelemetryEvent): void {
    this.events.push({ ...event, appId, timestamp: Date.now() });
  }

  flush(): Observable<void> {
    const batch = [...this.events];
    this.events = [];
    return this.sendBatch(batch);
  }
}
```

**Cand folosesti `providedIn: 'platform'`:**
- Micro-frontend: mai multe aplicatii Angular pe aceeasi pagina
- Analytics/Telemetry partajate
- Comunicare intre aplicatii

### Comparatie rapida

```
| Option       | Scope                          | Tree-shakable | Instante        |
|-------------|--------------------------------|---------------|-----------------|
| 'root'      | Aplicatia intreaga             | Da            | 1 singleton     |
| 'any'       | Per modul lazy-loaded          | Da            | 1 per modul     |
| 'platform'  | Toate app-urile de pe pagina   | Da            | 1 per platforma |
| NgModule    | Depinde de modul               | NU            | Variabil        |
| Component   | Per instanta de componenta     | N/A           | Per componenta  |
```

---

## Multi-providers

Multi-providers permit mai multor provideri sa contribuie valori sub acelasi token. Rezultatul este un array de toate valorile inregistrate. Acesta este mecanismul pe care Angular il foloseste intern pentru extensibilitate.

### Conceptul de baza

```typescript
const MY_HANDLERS = new InjectionToken<Handler[]>('my.handlers');

@NgModule({
  providers: [
    { provide: MY_HANDLERS, useClass: LoggingHandler, multi: true },
    { provide: MY_HANDLERS, useClass: MetricsHandler, multi: true },
    { provide: MY_HANDLERS, useClass: ErrorTrackingHandler, multi: true }
  ]
})
export class HandlersModule {}

// Injectarea returneaza un array
@Injectable({ providedIn: 'root' })
export class EventBus {
  private handlers = inject(MY_HANDLERS);

  emit(event: AppEvent): void {
    // handlers este Handler[] - toate cele 3 implementari
    for (const handler of this.handlers) {
      handler.handle(event);
    }
  }
}
```

### HTTP_INTERCEPTORS (pattern clasic, pre-functional)

```typescript
// Inainte de Angular 15+ functional interceptors
@NgModule({
  providers: [
    {
      provide: HTTP_INTERCEPTORS,
      useClass: AuthInterceptor,
      multi: true
    },
    {
      provide: HTTP_INTERCEPTORS,
      useClass: LoggingInterceptor,
      multi: true
    },
    {
      provide: HTTP_INTERCEPTORS,
      useClass: ErrorInterceptor,
      multi: true
    },
    {
      provide: HTTP_INTERCEPTORS,
      useClass: CacheInterceptor,
      multi: true
    }
  ]
})
export class HttpInterceptorModule {}

// Ordinea CONTEAZA - interceptorii sunt executati in ordinea inregistrarii
// Request:  Auth -> Logging -> Error -> Cache -> Server
// Response: Server -> Cache -> Error -> Logging -> Auth
```

### APP_INITIALIZER

Ruleaza logica asincrona inainte ca aplicatia sa se afiseze:

```typescript
// Functie factory care incarca configuratia de la server
function initializeAppConfig(
  http: HttpClient,
  config: AppConfigService
): () => Observable<void> {
  return () => http.get<AppConfig>('/api/config').pipe(
    tap(remoteConfig => config.setConfig(remoteConfig)),
    map(() => undefined)
  );
}

// Functie factory care incarca traducerile
function initializeTranslations(
  translate: TranslateService
): () => Promise<void> {
  return () => translate.loadLanguage(navigator.language).toPromise();
}

// Functie factory care verifica sesiunea utilizatorului
function initializeAuth(auth: AuthService): () => Observable<void> {
  return () => auth.checkExistingSession().pipe(
    catchError(() => of(undefined))  // Nu bloca aplicatia daca sesiunea a expirat
  );
}

@NgModule({
  providers: [
    {
      provide: APP_INITIALIZER,
      useFactory: initializeAppConfig,
      deps: [HttpClient, AppConfigService],
      multi: true
    },
    {
      provide: APP_INITIALIZER,
      useFactory: initializeTranslations,
      deps: [TranslateService],
      multi: true
    },
    {
      provide: APP_INITIALIZER,
      useFactory: initializeAuth,
      deps: [AuthService],
      multi: true
    }
  ]
})
export class AppModule {}

// NOTA: Toti APP_INITIALIZER-ii ruleaza in paralel.
// Aplicatia asteapta ca TOTI sa se termine inainte de bootstrap.
// Daca unul arunca eroare, aplicatia NU se incarca.
```

### NG_VALIDATORS si NG_ASYNC_VALIDATORS

```typescript
// Validator custom ca directiva cu multi-provider
@Directive({
  selector: '[appMinAge]',
  providers: [
    {
      provide: NG_VALIDATORS,
      useExisting: forwardRef(() => MinAgeValidatorDirective),
      multi: true
    }
  ]
})
export class MinAgeValidatorDirective implements Validator {
  @Input() appMinAge = 18;

  validate(control: AbstractControl): ValidationErrors | null {
    const birthDate = new Date(control.value);
    const age = this.calculateAge(birthDate);

    if (age < this.appMinAge) {
      return {
        minAge: {
          requiredAge: this.appMinAge,
          actualAge: age
        }
      };
    }
    return null;
  }

  private calculateAge(birthDate: Date): number {
    const today = new Date();
    let age = today.getFullYear() - birthDate.getFullYear();
    const monthDiff = today.getMonth() - birthDate.getMonth();
    if (monthDiff < 0 || (monthDiff === 0 && today.getDate() < birthDate.getDate())) {
      age--;
    }
    return age;
  }
}
```

### ENVIRONMENT_INITIALIZER (Angular 14+)

Multi-provider modern pentru standalone components:

```typescript
// Pattern modern cu standalone APIs
export function provideAnalytics(): EnvironmentProviders {
  return makeEnvironmentProviders([
    AnalyticsService,
    {
      provide: ENVIRONMENT_INITIALIZER,
      useValue: () => {
        const analytics = inject(AnalyticsService);
        const router = inject(Router);

        // Setup automat de page tracking
        router.events.pipe(
          filter(e => e instanceof NavigationEnd)
        ).subscribe(event => {
          analytics.trackPageView((event as NavigationEnd).urlAfterRedirects);
        });
      },
      multi: true
    }
  ]);
}

// Folosire in bootstrapApplication
bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes),
    provideHttpClient(withInterceptors([authInterceptor])),
    provideAnalytics()  // initializare automata
  ]
});
```

### Crearea propriilor extensii cu multi-providers

```typescript
// Definim un token pentru plugin-uri
export const APP_PLUGIN = new InjectionToken<AppPlugin[]>('app.plugins');

export interface AppPlugin {
  name: string;
  initialize(): void | Promise<void>;
  onEvent?(event: AppEvent): void;
  onDestroy?(): void;
}

// Plugin de logging
@Injectable()
export class LoggingPlugin implements AppPlugin {
  name = 'logging';

  initialize(): void {
    console.log('[LoggingPlugin] Initialized');
  }

  onEvent(event: AppEvent): void {
    console.log(`[LoggingPlugin] Event: ${event.type}`, event.payload);
  }
}

// Plugin de performance monitoring
@Injectable()
export class PerformancePlugin implements AppPlugin {
  name = 'performance';

  initialize(): void {
    if (typeof PerformanceObserver !== 'undefined') {
      const observer = new PerformanceObserver((list) => {
        for (const entry of list.getEntries()) {
          this.reportMetric(entry);
        }
      });
      observer.observe({ entryTypes: ['measure', 'navigation'] });
    }
  }

  private reportMetric(entry: PerformanceEntry): void {
    // Trimite metrici la backend
  }
}

// Inregistrare modulara
@NgModule({
  providers: [
    { provide: APP_PLUGIN, useClass: LoggingPlugin, multi: true },
    { provide: APP_PLUGIN, useClass: PerformancePlugin, multi: true }
  ]
})
export class PluginsModule {}

// Serviciul care consuma plugin-urile
@Injectable({ providedIn: 'root' })
export class PluginManagerService {
  private plugins = inject(APP_PLUGIN, { optional: true }) ?? [];

  async initializeAll(): Promise<void> {
    for (const plugin of this.plugins) {
      await plugin.initialize();
      console.log(`Plugin "${plugin.name}" initializat`);
    }
  }

  broadcast(event: AppEvent): void {
    for (const plugin of this.plugins) {
      plugin.onEvent?.(event);
    }
  }
}
```

---

## Tree-shakable providers

### Problema cu NgModule providers

Cand un serviciu este inregistrat in `providers` al unui NgModule, bundler-ul nu poate determina daca serviciul este folosit efectiv. Referinta este de la modul catre serviciu, iar modulul este intotdeauna inclus.

```typescript
// NU tree-shakable - serviciul va fi INTOTDEAUNA in bundle
@NgModule({
  providers: [
    RarelyUsedService,       // inclus chiar daca nimeni nu il injecteaza
    AnalyticsService,        // inclus chiar daca nimeni nu il injecteaza
    LegacyCompatService      // inclus chiar daca nimeni nu il injecteaza
  ]
})
export class SharedModule {}
```

### Cum functioneaza tree-shaking cu providedIn

Cu `providedIn: 'root'`, referinta este INVERSATA: serviciul stie despre injector, nu invers. Daca niciun cod nu importa serviciul, bundler-ul il elimina.

```typescript
// Tree-shakable - inclus DOAR daca cineva il injecteaza
@Injectable({
  providedIn: 'root'
})
export class AnalyticsService {
  // Daca nicio componenta/serviciu nu face inject(AnalyticsService),
  // intregul cod al acestei clase este eliminat din bundle
  track(event: string): void { /* ... */ }
}

// Tree-shakable cu InjectionToken
export const FEATURE_CONFIG = new InjectionToken<FeatureConfig>('feature.config', {
  providedIn: 'root',
  factory: () => ({
    enableBeta: false,
    maxItems: 100,
    theme: 'light'
  })
});
// Daca FEATURE_CONFIG nu este niciodata injectat, factory-ul si valoarea sunt eliminate
```

### Vizualizarea diferentei

```
// Referinta NgModule providers (NU tree-shakable):
// NgModule --imports--> ServiceA
// NgModule --imports--> ServiceB (nefolosit, dar INCLUS)
// NgModule --imports--> ServiceC (nefolosit, dar INCLUS)

// Referinta providedIn (tree-shakable):
// ServiceA --providedIn--> 'root'  (ComponentX il importa -> INCLUS)
// ServiceB --providedIn--> 'root'  (nimeni nu il importa -> ELIMINAT)
// ServiceC --providedIn--> 'root'  (nimeni nu il importa -> ELIMINAT)
```

### Impactul practic

```typescript
// Intr-o aplicatie enterprise cu ~200 servicii:
// - Cu NgModule providers: TOATE 200 sunt in bundle
// - Cu providedIn: 'root': doar cele ~80 efectiv folosite sunt in bundle

// Regula: Foloseste INTOTDEAUNA providedIn: 'root' pentru servicii noi
// Exceptii:
// 1. Servicii care trebuie sa fie per-componenta (foloseste component providers)
// 2. Servicii care necesita multi: true (trebuie NgModule sau makeEnvironmentProviders)
// 3. Servicii cu useClass/useFactory conditionat pe environment
```

### Pattern modern cu standalone: makeEnvironmentProviders

```typescript
// Functie provider tree-shakable cu configurare
export function provideFeatureFlags(config: FeatureFlagConfig): EnvironmentProviders {
  return makeEnvironmentProviders([
    {
      provide: FEATURE_FLAG_CONFIG,
      useValue: config
    },
    {
      provide: ENVIRONMENT_INITIALIZER,
      useValue: () => {
        const service = inject(FeatureFlagService);
        service.initialize(config);
      },
      multi: true
    }
  ]);
}

// Utilizare - tree-shakable: daca provideFeatureFlags nu este apelat,
// FeatureFlagService si tot codul asociat sunt eliminate din bundle
bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes),
    provideFeatureFlags({
      source: 'remote',
      refreshInterval: 60_000,
      defaultFlags: { darkMode: false }
    })
  ]
});
```

---

## inject() function vs constructor injection

### Constructor injection (modul traditional)

```typescript
@Component({
  selector: 'app-user-list',
  template: `...`
})
export class UserListComponent implements OnInit {
  users: User[] = [];

  constructor(
    private userService: UserService,
    private route: ActivatedRoute,
    private router: Router,
    @Optional() private analytics: AnalyticsService | null,
    @Inject(APP_CONFIG) private config: AppConfig
  ) {}

  ngOnInit(): void {
    this.userService.getAll().subscribe(users => {
      this.users = users;
      this.analytics?.track('users_loaded', { count: users.length });
    });
  }
}
```

### inject() function (modul modern - Angular 14+)

```typescript
@Component({
  selector: 'app-user-list',
  template: `...`
})
export class UserListComponent implements OnInit {
  private userService = inject(UserService);
  private route = inject(ActivatedRoute);
  private router = inject(Router);
  private analytics = inject(AnalyticsService, { optional: true });
  private config = inject<AppConfig>(APP_CONFIG);

  // Poate fi folosit direct in initializarea field-urilor
  private apiUrl = inject<AppConfig>(APP_CONFIG).apiBaseUrl;
  users = signal<User[]>([]);

  ngOnInit(): void {
    this.userService.getAll().subscribe(users => {
      this.users.set(users);
      this.analytics?.track('users_loaded', { count: users.length });
    });
  }
}
```

### Avantajele inject() fata de constructor injection

```typescript
// 1. COMPOSABILITATE - poti extrage logica de injectare in functii reutilizabile
// Aceasta este cel mai mare avantaj

function injectDestroy(): Observable<void> {
  const destroyRef = inject(DestroyRef);
  const subject = new Subject<void>();
  destroyRef.onDestroy(() => {
    subject.next();
    subject.complete();
  });
  return subject.asObservable();
}

function injectRouteParam(param: string): Signal<string | null> {
  const route = inject(ActivatedRoute);
  return toSignal(
    route.paramMap.pipe(map(params => params.get(param))),
    { initialValue: null }
  );
}

function injectCurrentUser(): Signal<User | null> {
  const auth = inject(AuthService);
  return auth.currentUser;
}

// Folosire in componente - extrem de curat
@Component({
  selector: 'app-user-profile',
  template: `
    @if (user(); as user) {
      <h1>{{ user.name }}</h1>
    }
  `
})
export class UserProfileComponent {
  private userId = injectRouteParam('id');
  private currentUser = injectCurrentUser();
  private destroy$ = injectDestroy();
  // Niciun constructor necesar!
}

// 2. MOSTENIREA ESTE SIMPLIFICATA
// Cu constructor injection:
class BaseComponent {
  constructor(
    protected router: Router,
    protected auth: AuthService
  ) {}
}

class ChildComponent extends BaseComponent {
  constructor(
    router: Router,
    auth: AuthService,
    private extra: ExtraService  // trebuie sa trimiti toate dependentele parintelui
  ) {
    super(router, auth);  // fragil, orice schimbare in parinte rupe copilul
  }
}

// Cu inject():
class BaseComponent {
  protected router = inject(Router);
  protected auth = inject(AuthService);
}

class ChildComponent extends BaseComponent {
  private extra = inject(ExtraService);
  // Nu trebuie constructor, nu trebuie super() cu argumente
}

// 3. TIPURI MAI BUNE - nu ai nevoie de @Inject() pentru token-uri
// Constructor: @Inject(APP_CONFIG) private config: AppConfig
// inject(): private config = inject<AppConfig>(APP_CONFIG);

// 4. CONDITIONAL/COMPUTED injection
@Component({ /* ... */ })
export class SmartComponent {
  private platformId = inject(PLATFORM_ID);
  private storage = isPlatformBrowser(this.platformId)
    ? inject(BrowserStorageService)
    : inject(ServerStorageService);
}
```

### inject() in functional guards, interceptors, resolvers

Angular 14+ permite guards, interceptors si resolvers ca simple functii, eliminand nevoia de clase:

```typescript
// Functional guard
export const authGuard: CanActivateFn = (route, state) => {
  const auth = inject(AuthService);
  const router = inject(Router);

  if (auth.isAuthenticated()) {
    return true;
  }

  return router.createUrlTree(['/login'], {
    queryParams: { returnUrl: state.url }
  });
};

// Guard cu role-based access
export const roleGuard = (requiredRole: string): CanActivateFn => {
  return () => {
    const auth = inject(AuthService);
    const router = inject(Router);

    if (auth.hasRole(requiredRole)) {
      return true;
    }

    return router.createUrlTree(['/unauthorized']);
  };
};

// Folosire in rute:
const routes: Routes = [
  {
    path: 'admin',
    canActivate: [authGuard, roleGuard('admin')],
    loadComponent: () => import('./admin.component')
  }
];

// Functional interceptor
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const auth = inject(AuthService);
  const token = auth.getToken();

  if (token) {
    req = req.clone({
      setHeaders: { Authorization: `Bearer ${token}` }
    });
  }

  return next(req);
};

// Functional interceptor cu retry si refresh token
export const tokenRefreshInterceptor: HttpInterceptorFn = (req, next) => {
  const auth = inject(AuthService);

  return next(req).pipe(
    catchError(error => {
      if (error.status === 401 && !req.url.includes('/refresh')) {
        return auth.refreshToken().pipe(
          switchMap(newToken => {
            const cloned = req.clone({
              setHeaders: { Authorization: `Bearer ${newToken}` }
            });
            return next(cloned);
          }),
          catchError(() => {
            auth.logout();
            return throwError(() => error);
          })
        );
      }
      return throwError(() => error);
    })
  );
};

// Functional resolver
export const userResolver: ResolveFn<User> = (route) => {
  const userService = inject(UserService);
  const id = route.paramMap.get('id')!;

  return userService.getById(id).pipe(
    catchError(() => {
      inject(Router).navigate(['/not-found']);
      return EMPTY;
    })
  );
};

// Inregistrare (standalone):
bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes),
    provideHttpClient(
      withInterceptors([authInterceptor, tokenRefreshInterceptor])
    )
  ]
});
```

### Regulile contextului de injectare

`inject()` poate fi apelat DOAR intr-un **injection context**. Acesta exista in:

```typescript
// 1. Constructor sau field initializer al unei clase cu decorator Angular
@Component({ /* ... */ })
export class MyComponent {
  private service = inject(MyService);  // VALID - field initializer
  constructor() {
    const router = inject(Router);      // VALID - constructor body
  }
}

// 2. Factory function al unui provider
export const MY_TOKEN = new InjectionToken('my.token', {
  providedIn: 'root',
  factory: () => {
    const http = inject(HttpClient);    // VALID - factory
    return new MyApiClient(http);
  }
});

// 3. Functii factory de provider (useFactory)
{
  provide: SomeService,
  useFactory: () => {
    const dep = inject(OtherService);   // VALID - useFactory
    return new SomeService(dep);
  }
}

// 4. Functional guards, interceptors, resolvers (runtime injection context)
export const myGuard: CanActivateFn = () => {
  const auth = inject(AuthService);     // VALID - functional guard
  return auth.isAuthenticated();
};

// 5. runInInjectionContext - pentru a crea manual un context
@Injectable({ providedIn: 'root' })
export class DynamicPluginLoader {
  private injector = inject(EnvironmentInjector);

  loadPlugin(pluginFactory: () => Plugin): Plugin {
    // Creaza un injection context manual
    return runInInjectionContext(this.injector, () => {
      return pluginFactory();  // inject() functioneaza in pluginFactory
    });
  }
}

// INVALID - inject() in afara contextului de injectare
@Component({ /* ... */ })
export class BadComponent {
  ngOnInit() {
    const service = inject(MyService);  // EROARE RUNTIME!
    // Error: inject() must be called from an injection context
  }

  onClick() {
    const router = inject(Router);      // EROARE RUNTIME!
  }
}

// Solutia cu runInInjectionContext pentru cazuri speciale:
@Component({ /* ... */ })
export class DynamicComponent {
  private injector = inject(EnvironmentInjector);

  loadDynamicFeature(): void {
    runInInjectionContext(this.injector, () => {
      const service = inject(DynamicService);  // Acum functioneaza
      service.initialize();
    });
  }
}
```

### Cand folosesti care abordare

```
| Criteriu                         | inject()    | Constructor          |
|----------------------------------|-------------|----------------------|
| Cod nou (Angular 14+)           | Preferat    | Acceptabil           |
| Functii reutilizabile (custom)  | Obligatoriu | Nu se poate          |
| Functional guards/interceptors  | Obligatoriu | Nu se aplica         |
| Mostenire de clase              | Preferat    | Fragil               |
| Cod legacy / migrare            | Optional    | Mentine consistenta  |
| Testare                         | Similar     | Similar              |
```

---

## Intrebari frecvente de interviu

### 1. Explica ierarhia de injectori in Angular si ordinea de rezolutie a dependentelor.

**Raspuns:** Angular are doua ierarhii paralele de injectori: **Element Injector** (creat per componenta/directiva) si **Module Injector** (creat per NgModule). Cand o dependenta este ceruta, Angular cauta mai intai in Element Injector-ul componentei curente, apoi urca in Element Injectors ai componentelor parinte prin DOM. Daca nu gaseste, trece la Module Injector al modulului care declara componenta, apoi urca prin Module Injectors parinte pana la Root Injector, Platform Injector, si in final NullInjector care arunca eroare (sau returneaza null cu `@Optional()`). Aceasta ierarhie duala permite atat izolarea serviciilor per componenta, cat si partajarea globala.

### 2. Care este diferenta dintre `providers` si `viewProviders` la nivel de componenta?

**Raspuns:** `providers` inregistreaza servicii in Element Injector-ul componentei, facandu-le vizibile pentru toti copiii: atat cei din template (view children), cat si cei din content projection (`<ng-content>`). `viewProviders` limiteaza vizibilitatea DOAR la copiii din template. Continutul proiectat NU poate accesa serviciile din `viewProviders`. Acest mecanism este util pentru a preveni scurgerea de dependente interne catre componente externe care sunt proiectate in componenta curenta. Un exemplu clasic este un component library care nu vrea sa expuna servicii interne catre continutul proiectat de consumator.

### 3. Cand si de ce ai folosi `providedIn: 'any'` in loc de `providedIn: 'root'`?

**Raspuns:** `providedIn: 'root'` creaza un singleton global, pe cand `providedIn: 'any'` creaza o instanta separata pentru fiecare lazy-loaded module (modulele eager-loaded impart aceeasi instanta cu root). Folosesti `'any'` cand ai nevoie de state izolat per feature module: de exemplu, un cache service care trebuie sa fie independent intre sectiunile Orders si Products ale aplicatiei, fiecare cu propriile date cached. In practica, `'any'` este rar folosit; de cele mai multe ori, component-level providers ofera un control mai explicit si mai usor de inteles.

### 4. Cum functioneaza tree-shaking-ul cu `providedIn: 'root'` comparativ cu NgModule providers?

**Raspuns:** Cu NgModule providers, referinta merge de la modul catre serviciu: modulul "stie" despre serviciu. Deoarece modulul este intotdeauna inclus in bundle, si serviciul va fi inclus, chiar daca nimeni nu il injecteaza. Cu `providedIn: 'root'`, referinta este inversata: serviciul "stie" despre injector prin decoratorul `@Injectable`. Daca niciun cod din aplicatie nu face `import` la serviciu si nu il injecteaza, bundler-ul (webpack/esbuild) determina ca serviciul este "dead code" si il elimina din bundle. Aceasta poate reduce semnificativ dimensiunea bundle-ului in aplicatii enterprise cu multe servicii.

### 5. Explica diferenta dintre `@Self()`, `@SkipSelf()` si `@Host()` cu exemple practice.

**Raspuns:** `@Self()` restrictioneaza cautarea la injectorul curent al componentei/directivei - util cand vrei sa garantezi ca folosesti o instanta locala, nu una mostenita accidental. `@SkipSelf()` sare peste injectorul curent si cauta doar in parinti - pattern-ul clasic pentru structuri ierarhice (ex: un TreeNodeService unde fiecare nod trebuie sa gaseasca parintele, nu pe sine). `@Host()` opreste cautarea la componenta gazda - relevant in directivele aplicate pe componente, unde nu vrei sa "scapi" peste granita host-ului. Combinatia `@SkipSelf() @Optional()` este pattern-ul standard pentru radacina unei ierarhii: nodul radacina nu are parinte, deci primeste `null`.

### 6. Ce este `forwardRef()` si de ce mai este necesar in Angular modern?

**Raspuns:** `forwardRef()` amana rezolutia unei referinte pana la runtime, rezolvand problema ordinii de declarare in JavaScript (o clasa nu poate fi referita inainte de declarare). In Angular modern, cel mai comun caz ramane pattern-ul `NG_VALUE_ACCESSOR` si `NG_VALIDATORS` cu `useExisting`: decoratorul `@Component` este evaluat inainte ca clasa sa fie complet definita, deci `useExisting: MyComponent` ar esua fara `forwardRef(() => MyComponent)`. Pentru dependentele circulare intre servicii, restructurarea codului (extractia intr-un al treilea serviciu mediator) este de preferat fata de `forwardRef()`.

### 7. Cum functioneaza multi-providers si care sunt cazurile de utilizare principale?

**Raspuns:** Multi-providers permit mai multor provideri sa contribuie valori sub acelasi token, iar rezultatul injectarii este un array cu toate valorile. Se activeaza cu `multi: true`. Cazurile principale sunt: `HTTP_INTERCEPTORS` (chain of responsibility pentru request/response), `APP_INITIALIZER` (task-uri de initializare executate in paralel inainte de bootstrap), `NG_VALIDATORS` si `NG_ASYNC_VALIDATORS` (validatoare compuse pe un form control), si `ENVIRONMENT_INITIALIZER` (setup logic in standalone apps). Pattern-ul este excelent pentru extensibilitate: permite ca module separate sa contribuie la un comportament comun fara sa se cunoasca intre ele (open-closed principle).

### 8. Care sunt regulile pentru injection context si unde poti folosi `inject()`?

**Raspuns:** `inject()` functioneaza DOAR intr-un injection context, care exista in: (1) constructorul si field initializers ale claselor Angular (`@Component`, `@Directive`, `@Injectable`, `@Pipe`), (2) functii factory ale `InjectionToken` (`providedIn` + `factory`), (3) `useFactory` al providerilor, (4) functional guards, interceptors, resolvers (Angular seteaza contextul la runtime), (5) callback-uri `ENVIRONMENT_INITIALIZER`. In AFARA acestor contexte (ngOnInit, event handlers, subscribe callbacks), `inject()` va arunca o eroare runtime. Pentru cazuri speciale, `runInInjectionContext(injector, fn)` permite crearea manuala a unui context.

### 9. Cum ai proiecta un sistem de DI pentru o aplicatie enterprise cu multiple echipe?

**Raspuns:** Principii cheie: (1) Servicii globale cu `providedIn: 'root'` pentru singletons (AuthService, ConfigService, HttpClient wrappers). (2) Servicii feature-scoped cu component-level providers pentru state izolat per feature. (3) InjectionTokens pentru configuratii, nu clase concrete - permite fiecarei echipe sa injecteze propria configuratie. (4) Abstract classes ca token-uri pentru servicii cu implementari multiple (LoggerService abstract, cu CloudLogger sau ConsoleLogger). (5) `makeEnvironmentProviders()` cu functii `provideXxx()` pentru a crea API-uri curate de configurare: `provideFeatureFlags({...})`, `provideAnalytics({...})`. (6) Multi-providers pentru extensibilitate: un token PLUGIN la care fiecare echipa contribuie. (7) Evitarea NgModule providers in favoarea tree-shakable alternatives.

### 10. Compara `inject()` function cu constructor injection. Cand alegi una fata de cealalta?

**Raspuns:** `inject()` este superior in mai multe privinte: permite crearea de **custom injection functions** reutilizabile (ex: `injectDestroy()`, `injectRouteParam()`), simplifica mostenirea (nu mai este nevoie de `super()` cu toate dependentele parintelui), ofera tipuri mai bune fara `@Inject()`, si este obligatoriu pentru functional guards/interceptors/resolvers. Constructor injection ramane valid pentru cod legacy si cand echipa prefera consistenta cu codul existent. Regula practica: cod nou foloseste `inject()`, cod existent se migreaza gradual. Singura limitare a `inject()` este ca trebuie apelat in injection context - nu poate fi folosit in metode de lifecycle sau event handlers fara `runInInjectionContext()`.
