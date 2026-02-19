# 06 - Testare in Angular

## Cuprins
1. [Strategii de testare (piramida testelor)](#1-strategii-de-testare)
2. [Unit testing cu Jest](#2-unit-testing-cu-jest)
3. [TestBed si testing utilities Angular](#3-testbed-si-testing-utilities)
4. [Testing components](#4-testing-components)
5. [Testing services](#5-testing-services)
6. [Testing pipes si directives](#6-testing-pipes-si-directives)
7. [Testing formulare reactive](#7-testing-formulare-reactive)
8. [Mocking dependencies](#8-mocking-dependencies)
9. [Integration testing](#9-integration-testing)
10. [E2E testing cu Cypress / Playwright](#10-e2e-testing)
11. [Testing best practices](#11-testing-best-practices)
12. [Code coverage si metrici](#12-code-coverage-si-metrici)
13. [Intrebari frecvente de interviu](#intrebari-frecvente-de-interviu)

---

## 1. Strategii de testare

### Piramida testelor (Test Pyramid - Mike Cohn)

```
        /  E2E  \          <-- Putine, lente, fragile, costisitoare
       /----------\
      / Integration \      <-- Moderate ca numar
     /----------------\
    /    Unit Tests     \  <-- Multe, rapide, ieftine, stabile
   /____________________\
```

**Unit Tests (baza piramidei):**
- Cele mai multe teste (70-80% din total)
- Testeaza o singura unitate izolata (functie, clasa, pipe, service)
- Rapide (milisecunde), stabile, usor de debugat
- Mock-uiesc toate dependintele externe

**Integration Tests (mijlocul piramidei):**
- Numar moderat (15-20%)
- Testeaza interactiunea intre 2+ unitati (component + service, component + child components)
- Mai lente decat unit tests, dar mult mai rapide decat E2E
- Mock-uiesc doar granita sistemului (HTTP, localStorage)

**E2E Tests (varful piramidei):**
- Putine (5-10%)
- Testeaza flow-uri complete din perspectiva utilizatorului
- Lente, fragile, costisitoare de mentinut
- Ruleaza in browser real, fara mocks (sau cu API mocks minimale)

### Testing Trophy (Kent C. Dodds)

```
        /  E2E  \
       /----------\
      /             \
     / Integration   \     <-- ACCENTUL principal aici
    /   Tests         \
   /-------------------\
    \  Unit  /  Static /   <-- Static = TypeScript, ESLint
     \______/________/
```

Diferenta fata de piramida clasica: Kent C. Dodds pune accent pe **integration tests** ca oferind cel mai bun raport cost/beneficiu. In Angular, asta inseamna sa testezi componente cu serviciile lor reale (dar cu HTTP mock-uit).

### Ce sa testezi la fiecare nivel

| Nivel | CE sa testezi | CE NU |
|-------|--------------|-------|
| **Unit** | Logica de business, pipes, validators, utility functions, services izolate | Template rendering, CSS, integrari externe |
| **Integration** | Component + children, component + service, router guards cu router | Stiluri vizuale, third-party libs internals |
| **E2E** | Critical user journeys, happy paths, authentication flows | Fiecare edge case, fiecare pagina |

---

## 2. Unit testing cu Jest

### De ce Jest in loc de Karma/Jasmine

| Criteriu | Karma/Jasmine | Jest |
|----------|--------------|------|
| **Viteza** | Lent (deschide browser) | Rapid (jsdom, no browser) |
| **Watch mode** | Reruleaza tot | Reruleaza doar testele afectate |
| **Snapshot testing** | Nu | Da, built-in |
| **Mocking** | Manual sau jasmine.createSpyObj | jest.fn(), jest.mock(), jest.spyOn() |
| **Paralelism** | Nu nativ | Da, teste ruleaza in paralel |
| **Configurare** | Default Angular CLI | Necesita setup initial |
| **Adoption** | In scadere | Standard industrie |

> **Nota:** Incepand cu Angular 16+, echipa Angular a inceput sa ofere suport experimental pentru Jest. Angular 17+ include builder nativ pentru Jest.

### Configurare Jest cu Angular

**Varianta 1: @angular-builders/jest (recomandat pentru proiecte existente)**

```bash
ng add @angular-builders/jest
```

Sau manual:

```bash
npm install -D jest @angular-builders/jest @types/jest
```

```json
// angular.json
{
  "projects": {
    "my-app": {
      "architect": {
        "test": {
          "builder": "@angular-builders/jest:run",
          "options": {
            "configPath": "./jest.config.ts"
          }
        }
      }
    }
  }
}
```

```typescript
// jest.config.ts
import type { Config } from 'jest';

const config: Config = {
  preset: 'jest-preset-angular',
  setupFilesAfterSetup: ['<rootDir>/setup-jest.ts'],
  testPathIgnorePatterns: ['<rootDir>/node_modules/', '<rootDir>/dist/'],
  coverageDirectory: 'coverage',
  coverageReporters: ['html', 'text-summary', 'lcov'],
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.module.ts',
    '!src/main.ts',
    '!src/**/*.d.ts'
  ]
};

export default config;
```

```typescript
// setup-jest.ts
import 'jest-preset-angular/setup-jest';
```

**Varianta 2: jest-preset-angular (standalone)**

```bash
npm install -D jest jest-preset-angular @types/jest ts-jest
```

### Structura de baza Jest

```typescript
// calculator.service.spec.ts
import { CalculatorService } from './calculator.service';

describe('CalculatorService', () => {
  let service: CalculatorService;

  // Ruleaza inainte de FIECARE test
  beforeEach(() => {
    service = new CalculatorService();
  });

  // Ruleaza dupa FIECARE test
  afterEach(() => {
    // cleanup daca e necesar
  });

  // Grup de teste inrudite
  describe('add', () => {
    it('should add two positive numbers', () => {
      expect(service.add(2, 3)).toBe(5);
    });

    it('should handle negative numbers', () => {
      expect(service.add(-1, -2)).toBe(-3);
    });
  });

  describe('divide', () => {
    it('should divide correctly', () => {
      expect(service.divide(10, 2)).toBe(5);
    });

    it('should throw on division by zero', () => {
      expect(() => service.divide(10, 0)).toThrow('Division by zero');
    });
  });
});
```

### Jest Matchers - referinta completa

```typescript
// Egalitate
expect(value).toBe(5);                    // === (referinta identica)
expect(obj).toEqual({ a: 1, b: 2 });     // deep equality (recomandatpentru obiecte)
expect(obj).toStrictEqual({ a: 1 });      // deep + verifica si proprietati undefined

// Truthiness
expect(value).toBeTruthy();
expect(value).toBeFalsy();
expect(value).toBeNull();
expect(value).toBeUndefined();
expect(value).toBeDefined();

// Numere
expect(value).toBeGreaterThan(3);
expect(value).toBeGreaterThanOrEqual(3);
expect(value).toBeLessThan(5);
expect(0.1 + 0.2).toBeCloseTo(0.3);      // floating point

// Stringuri
expect(str).toMatch(/pattern/);
expect(str).toContain('substring');

// Arrays
expect(arr).toContain(item);
expect(arr).toContainEqual({ id: 1 });    // deep equality in array
expect(arr).toHaveLength(3);

// Exceptii
expect(() => fn()).toThrow();
expect(() => fn()).toThrow('mesaj specific');
expect(() => fn()).toThrow(CustomError);

// Spy / Mock
expect(mockFn).toHaveBeenCalled();
expect(mockFn).toHaveBeenCalledTimes(2);
expect(mockFn).toHaveBeenCalledWith('arg1', expect.any(Number));
expect(mockFn).toHaveBeenLastCalledWith('lastArg');
expect(mockFn).toHaveReturnedWith(42);

// Asymmetric matchers (flexibilitate)
expect(obj).toEqual(expect.objectContaining({ name: 'test' }));
expect(arr).toEqual(expect.arrayContaining([1, 2]));
expect(str).toEqual(expect.stringContaining('partial'));
expect(val).toEqual(expect.any(String));
```

### Jest Mocking

```typescript
// jest.fn() - creaza o functie mock de la zero
const mockCallback = jest.fn();
mockCallback.mockReturnValue(42);
mockCallback.mockReturnValueOnce(1).mockReturnValueOnce(2);
mockCallback.mockImplementation((x: number) => x * 2);
mockCallback.mockResolvedValue('async result');     // pentru Promises
mockCallback.mockRejectedValue(new Error('fail'));  // Promise reject

// jest.spyOn() - spy pe o metoda existenta (pastreaza sau inlocuieste implementarea)
const service = new UserService();
const spy = jest.spyOn(service, 'getUser');
spy.mockReturnValue({ id: 1, name: 'Mock User' });
// ... test ...
expect(spy).toHaveBeenCalledWith(1);
spy.mockRestore();  // restaureaza implementarea originala

// jest.mock() - mock-uieste un modul intreg
jest.mock('./user.service', () => ({
  UserService: jest.fn().mockImplementation(() => ({
    getUser: jest.fn().mockReturnValue({ id: 1, name: 'Mock' }),
    saveUser: jest.fn().mockResolvedValue(true)
  }))
}));

// Mock partial (pastreaza restul modulului real)
jest.mock('./utils', () => ({
  ...jest.requireActual('./utils'),
  formatDate: jest.fn().mockReturnValue('2024-01-01')
}));
```

---

## 3. TestBed si testing utilities

### TestBed.configureTestingModule()

TestBed este cel mai important utility Angular pentru teste. Creeaza un modul Angular de test care simuleaza un `NgModule`.

```typescript
import { TestBed, ComponentFixture } from '@angular/core/testing';
import { UserListComponent } from './user-list.component';
import { UserService } from './user.service';
import { HttpClientTestingModule } from '@angular/common/http/testing';

describe('UserListComponent', () => {
  let component: UserListComponent;
  let fixture: ComponentFixture<UserListComponent>;
  let userService: UserService;

  beforeEach(async () => {
    // configureTestingModule e ASINCRON (compileaza componentele)
    await TestBed.configureTestingModule({
      // Pentru standalone components (Angular 14+):
      imports: [
        UserListComponent,        // componenta standalone se importa
        HttpClientTestingModule
      ],
      // Pentru module-based components:
      // declarations: [UserListComponent],
      // imports: [HttpClientTestingModule],
      providers: [
        UserService,
        // sau mock: { provide: UserService, useValue: mockUserService }
      ]
    }).compileComponents();  // compileaza template-uri si stiluri

    fixture = TestBed.createComponent(UserListComponent);
    component = fixture.componentInstance;
    userService = TestBed.inject(UserService);
  });

  it('should create', () => {
    expect(component).toBeTruthy();
  });
});
```

### ComponentFixture si DebugElement

```typescript
// ComponentFixture<T> - wrapper in jurul componentei
const fixture = TestBed.createComponent(MyComponent);

// Instanta componentei
const component = fixture.componentInstance;

// Trigger change detection manual
fixture.detectChanges();  // OBLIGATORIU dupa modificari

// Acces la DOM nativ
const nativeElement: HTMLElement = fixture.nativeElement;
const title = nativeElement.querySelector('h1')?.textContent;

// DebugElement - wrapper Angular peste DOM
const debugEl: DebugElement = fixture.debugElement;

// Cautare cu By.css() - similar querySelector dar returneaza DebugElement
import { By } from '@angular/platform-browser';

const button = debugEl.query(By.css('button.submit'));
const allItems = debugEl.queryAll(By.css('.list-item'));

// Cautare cu By.directive()
const childComponent = debugEl.query(By.directive(ChildComponent));
const childInstance = childComponent.componentInstance as ChildComponent;

// Proprietati DebugElement
debugEl.nativeElement;       // elementul DOM nativ
debugEl.componentInstance;   // instanta componentei (daca e component element)
debugEl.properties;          // property bindings
debugEl.attributes;          // atribute statice
debugEl.classes;             // CSS classes
debugEl.styles;              // inline styles
debugEl.children;            // child DebugElements

// Trigger events pe DebugElement
button.triggerEventHandler('click', null);
// sau pe native element:
(button.nativeElement as HTMLButtonElement).click();
fixture.detectChanges();
```

### fakeAsync, tick, flush

Controleaza timpul in teste - esential pentru operatii asincrone (setTimeout, interval, debounce, Observable delays).

```typescript
import { fakeAsync, tick, flush } from '@angular/core/testing';

describe('AsyncComponent', () => {
  it('should load data after delay', fakeAsync(() => {
    const component = fixture.componentInstance;
    component.loadData();  // intern face setTimeout(() => ..., 1000)

    // Timpul NU trece automat in fakeAsync
    expect(component.data).toBeUndefined();

    // Avanseaza 1000ms
    tick(1000);
    expect(component.data).toBeDefined();

    fixture.detectChanges();
    expect(fixture.nativeElement.textContent).toContain('loaded');
  }));

  it('should handle debounced search', fakeAsync(() => {
    const input = fixture.debugElement.query(By.css('input'));
    input.nativeElement.value = 'angular';
    input.nativeElement.dispatchEvent(new Event('input'));

    // Debounce de 300ms
    tick(300);
    fixture.detectChanges();

    expect(component.searchResults.length).toBeGreaterThan(0);
  }));

  it('should complete all pending async', fakeAsync(() => {
    component.startMultipleTimers();

    // flush() - avanseaza pana se termina TOATE setTimeout/setInterval
    flush();

    expect(component.allDone).toBe(true);
  }));

  // discardPeriodicTasks - pentru setInterval care nu se termina
  it('should handle intervals', fakeAsync(() => {
    component.startPolling();  // setInterval la fiecare 5s
    tick(15000);               // 3 poll-uri
    expect(component.pollCount).toBe(3);
    discardPeriodicTasks();    // curata interval-ul ramas
  }));
});
```

### waitForAsync

Pentru teste cu Promises reale (nu simulate). Mai rar folosit decat fakeAsync.

```typescript
import { waitForAsync } from '@angular/core/testing';

it('should fetch data', waitForAsync(() => {
  // waitForAsync intercepteaza Zone.js si asteapta toate Promises
  component.fetchData().then(() => {
    fixture.detectChanges();
    expect(component.data).toBeDefined();
  });
}));
```

### HttpClientTestingModule si HttpTestingController

```typescript
import { HttpClientTestingModule, HttpTestingController } from '@angular/common/http/testing';

describe('UserService', () => {
  let service: UserService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [UserService]
    });

    service = TestBed.inject(UserService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    // Verifica ca nu exista request-uri neasteptate
    httpMock.verify();
  });

  it('should fetch users', () => {
    const mockUsers = [
      { id: 1, name: 'Alice' },
      { id: 2, name: 'Bob' }
    ];

    service.getUsers().subscribe(users => {
      expect(users).toEqual(mockUsers);
      expect(users.length).toBe(2);
    });

    // Captureaza request-ul si raspunde cu mock data
    const req = httpMock.expectOne('/api/users');
    expect(req.request.method).toBe('GET');
    expect(req.request.headers.get('Authorization')).toBeTruthy();
    req.flush(mockUsers);  // trimite raspunsul mock
  });

  it('should handle HTTP errors', () => {
    service.getUsers().subscribe({
      next: () => fail('should have failed'),
      error: (err) => {
        expect(err.status).toBe(500);
      }
    });

    const req = httpMock.expectOne('/api/users');
    req.flush('Server Error', {
      status: 500,
      statusText: 'Internal Server Error'
    });
  });

  it('should send POST with body', () => {
    const newUser = { name: 'Charlie', email: 'charlie@test.com' };

    service.createUser(newUser).subscribe(result => {
      expect(result.id).toBeDefined();
    });

    const req = httpMock.expectOne('/api/users');
    expect(req.request.method).toBe('POST');
    expect(req.request.body).toEqual(newUser);
    req.flush({ id: 3, ...newUser });
  });

  it('should match multiple requests', () => {
    service.getUsersByRole('admin').subscribe();
    service.getUsersByRole('user').subscribe();

    const requests = httpMock.match(req =>
      req.url.includes('/api/users') && req.params.has('role')
    );
    expect(requests.length).toBe(2);
    requests.forEach(req => req.flush([]));
  });
});
```

---

## 4. Testing components

### Shallow testing cu NO_ERRORS_SCHEMA

Testeaza componenta izolat, fara a randa componentele copil.

```typescript
import { NO_ERRORS_SCHEMA } from '@angular/core';

beforeEach(async () => {
  await TestBed.configureTestingModule({
    imports: [UserProfileComponent],
    schemas: [NO_ERRORS_SCHEMA]  // ignora elemente necunoscute (child components)
  }).compileComponents();
});

// Avantaje: rapid, izolat, nu trebuie sa importi child components
// Dezavantaje: nu detecteaza erori de binding pe child components
```

**Alternativa mai sigura - CUSTOM_ELEMENTS_SCHEMA:**

```typescript
import { CUSTOM_ELEMENTS_SCHEMA } from '@angular/core';
// Similar, dar permite doar custom elements (cu dash in nume: app-child)
```

### Testing @Input si @Output

```typescript
@Component({
  selector: 'app-counter',
  standalone: true,
  template: `
    <span class="count">{{ count }}</span>
    <button (click)="increment()">+</button>
    <button (click)="reset.emit()">Reset</button>
  `
})
export class CounterComponent {
  @Input() count = 0;
  @Output() countChange = new EventEmitter<number>();
  @Output() reset = new EventEmitter<void>();

  increment() {
    this.count++;
    this.countChange.emit(this.count);
  }
}

// Test
describe('CounterComponent', () => {
  let component: CounterComponent;
  let fixture: ComponentFixture<CounterComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [CounterComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(CounterComponent);
    component = fixture.componentInstance;
  });

  it('should display input count', () => {
    component.count = 5;
    fixture.detectChanges();

    const span = fixture.nativeElement.querySelector('.count');
    expect(span.textContent).toBe('5');
  });

  it('should emit countChange on increment', () => {
    component.count = 10;
    jest.spyOn(component.countChange, 'emit');

    component.increment();

    expect(component.countChange.emit).toHaveBeenCalledWith(11);
  });

  it('should emit reset when reset button clicked', () => {
    jest.spyOn(component.reset, 'emit');
    fixture.detectChanges();

    const resetBtn = fixture.debugElement.queryAll(By.css('button'))[1];
    resetBtn.triggerEventHandler('click', null);

    expect(component.reset.emit).toHaveBeenCalled();
  });
});
```

### Testing template rendering

```typescript
@Component({
  selector: 'app-user-card',
  standalone: true,
  imports: [NgIf, NgFor],
  template: `
    <div class="card" *ngIf="user">
      <h2 class="name">{{ user.name }}</h2>
      <span class="role" [class.admin]="user.role === 'admin'">
        {{ user.role }}
      </span>
      <ul>
        <li *ngFor="let skill of user.skills" class="skill">{{ skill }}</li>
      </ul>
    </div>
    <div class="no-user" *ngIf="!user">No user selected</div>
  `
})
export class UserCardComponent {
  @Input() user: User | null = null;
}

describe('UserCardComponent', () => {
  let fixture: ComponentFixture<UserCardComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [UserCardComponent]
    }).compileComponents();
    fixture = TestBed.createComponent(UserCardComponent);
  });

  it('should show "No user selected" when user is null', () => {
    fixture.detectChanges();
    const el = fixture.nativeElement;
    expect(el.querySelector('.no-user')).toBeTruthy();
    expect(el.querySelector('.card')).toBeNull();
  });

  it('should display user details', () => {
    fixture.componentInstance.user = {
      name: 'Alice',
      role: 'admin',
      skills: ['Angular', 'TypeScript']
    };
    fixture.detectChanges();

    const el = fixture.nativeElement;
    expect(el.querySelector('.name').textContent).toBe('Alice');
    expect(el.querySelector('.role').textContent.trim()).toBe('admin');
    expect(el.querySelector('.role').classList).toContain('admin');

    const skills = el.querySelectorAll('.skill');
    expect(skills.length).toBe(2);
    expect(skills[0].textContent).toBe('Angular');
  });
});
```

### Testing cu signal inputs (Angular 17+)

```typescript
@Component({
  selector: 'app-greeting',
  standalone: true,
  template: `<h1>Hello, {{ name() }}!</h1>`
})
export class GreetingComponent {
  name = input<string>('World');             // signal input cu default
  required = input.required<string>();        // signal input required
}

describe('GreetingComponent', () => {
  it('should use default signal input value', () => {
    const fixture = TestBed.createComponent(GreetingComponent);
    // Signal inputs cu required nu pot fi create direct cu TestBed.createComponent
    // Trebuie un wrapper sau componentInputs

    fixture.detectChanges();
    expect(fixture.nativeElement.textContent).toContain('Hello, World!');
  });

  // Varianta recomandata Angular 17.1+: fixture.componentRef.setInput()
  it('should update signal input', () => {
    const fixture = TestBed.createComponent(GreetingComponent);

    fixture.componentRef.setInput('name', 'Angular');
    fixture.detectChanges();

    expect(fixture.nativeElement.textContent).toContain('Hello, Angular!');
  });

  // Varianta cu host component pentru required inputs
  @Component({
    standalone: true,
    imports: [GreetingComponent],
    template: `<app-greeting [name]="testName" />`
  })
  class HostComponent {
    testName = 'Test';
  }

  it('should work with host component', () => {
    const hostFixture = TestBed.configureTestingModule({
      imports: [HostComponent]
    }).createComponent(HostComponent);

    hostFixture.detectChanges();
    const greeting = hostFixture.debugElement.query(By.directive(GreetingComponent));
    expect(greeting.nativeElement.textContent).toContain('Hello, Test!');
  });
});
```

### Component Harnesses (Angular CDK)

Component harnesses ofera un API stabil pentru testarea componentelor, abstractizand detaliile DOM.

```typescript
import { HarnessLoader } from '@angular/cdk/testing';
import { TestbedHarnessEnvironment } from '@angular/cdk/testing/testbed';
import { MatButtonHarness } from '@angular/material/button/testing';
import { MatInputHarness } from '@angular/material/input/testing';
import { MatSelectHarness } from '@angular/material/select/testing';

describe('FormComponent with Harnesses', () => {
  let loader: HarnessLoader;
  let fixture: ComponentFixture<MyFormComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [MyFormComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(MyFormComponent);
    loader = TestbedHarnessEnvironment.loader(fixture);
  });

  it('should fill and submit form', async () => {
    // Gaseste input-urile prin harness
    const nameInput = await loader.getHarness(
      MatInputHarness.with({ selector: '[formControlName="name"]' })
    );
    const roleSelect = await loader.getHarness(MatSelectHarness);
    const submitBtn = await loader.getHarness(
      MatButtonHarness.with({ text: 'Submit' })
    );

    // Interactioneaza
    await nameInput.setValue('Alice');
    await roleSelect.open();
    await roleSelect.clickOptions({ text: 'Admin' });

    expect(await submitBtn.isDisabled()).toBe(false);
    await submitBtn.click();

    expect(fixture.componentInstance.submitted).toBe(true);
  });

  // Crearea unui custom harness
  // user-card.harness.ts
  // import { ComponentHarness } from '@angular/cdk/testing';
  //
  // export class UserCardHarness extends ComponentHarness {
  //   static hostSelector = 'app-user-card';
  //
  //   private getNameEl = this.locatorFor('.name');
  //   private getRoleEl = this.locatorFor('.role');
  //   private getEditBtn = this.locatorForOptional('button.edit');
  //
  //   async getName(): Promise<string> {
  //     return (await this.getNameEl()).text();
  //   }
  //   async getRole(): Promise<string> {
  //     return (await this.getRoleEl()).text();
  //   }
  //   async clickEdit(): Promise<void> {
  //     const btn = await this.getEditBtn();
  //     if (btn) await btn.click();
  //   }
  // }
});
```

### Exemplu complet: testing a form component

```typescript
// login-form.component.ts
@Component({
  selector: 'app-login-form',
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <input formControlName="email" data-testid="email" />
      <div class="error" *ngIf="form.get('email')?.errors?.['email']">
        Invalid email
      </div>
      <input formControlName="password" type="password" data-testid="password" />
      <div class="error" *ngIf="form.get('password')?.errors?.['minlength']">
        Password too short
      </div>
      <button type="submit" [disabled]="form.invalid">Login</button>
      <div class="server-error" *ngIf="serverError">{{ serverError }}</div>
    </form>
  `
})
export class LoginFormComponent {
  form = new FormGroup({
    email: new FormControl('', [Validators.required, Validators.email]),
    password: new FormControl('', [Validators.required, Validators.minLength(8)])
  });
  serverError = '';

  constructor(private authService: AuthService) {}

  onSubmit() {
    if (this.form.valid) {
      this.authService.login(this.form.value).subscribe({
        next: () => { /* redirect */ },
        error: (err) => this.serverError = err.message
      });
    }
  }
}

// login-form.component.spec.ts
describe('LoginFormComponent', () => {
  let fixture: ComponentFixture<LoginFormComponent>;
  let component: LoginFormComponent;
  let authService: jest.Mocked<AuthService>;

  beforeEach(async () => {
    const mockAuthService = {
      login: jest.fn()
    };

    await TestBed.configureTestingModule({
      imports: [LoginFormComponent],
      providers: [
        { provide: AuthService, useValue: mockAuthService }
      ]
    }).compileComponents();

    fixture = TestBed.createComponent(LoginFormComponent);
    component = fixture.componentInstance;
    authService = TestBed.inject(AuthService) as jest.Mocked<AuthService>;
    fixture.detectChanges();
  });

  it('should start with invalid form', () => {
    expect(component.form.valid).toBe(false);
  });

  it('should validate email format', () => {
    component.form.patchValue({ email: 'invalid' });
    expect(component.form.get('email')?.errors?.['email']).toBeTruthy();

    component.form.patchValue({ email: 'valid@test.com' });
    expect(component.form.get('email')?.errors).toBeNull();
  });

  it('should validate password minimum length', () => {
    component.form.patchValue({ password: '123' });
    expect(component.form.get('password')?.errors?.['minlength']).toBeTruthy();

    component.form.patchValue({ password: '12345678' });
    expect(component.form.get('password')?.errors).toBeNull();
  });

  it('should disable submit button when form is invalid', () => {
    fixture.detectChanges();
    const btn = fixture.nativeElement.querySelector('button');
    expect(btn.disabled).toBe(true);
  });

  it('should enable submit button when form is valid', () => {
    component.form.patchValue({
      email: 'test@example.com',
      password: 'password123'
    });
    fixture.detectChanges();

    const btn = fixture.nativeElement.querySelector('button');
    expect(btn.disabled).toBe(false);
  });

  it('should call authService.login on valid submit', () => {
    authService.login.mockReturnValue(of({ token: 'abc' }));

    component.form.patchValue({
      email: 'test@example.com',
      password: 'password123'
    });
    component.onSubmit();

    expect(authService.login).toHaveBeenCalledWith({
      email: 'test@example.com',
      password: 'password123'
    });
  });

  it('should display server error on login failure', () => {
    authService.login.mockReturnValue(
      throwError(() => new Error('Invalid credentials'))
    );

    component.form.patchValue({
      email: 'test@example.com',
      password: 'wrongpassword'
    });
    component.onSubmit();
    fixture.detectChanges();

    const errorEl = fixture.nativeElement.querySelector('.server-error');
    expect(errorEl.textContent).toContain('Invalid credentials');
  });

  it('should show email validation error in template', () => {
    component.form.get('email')?.setValue('invalid');
    component.form.get('email')?.markAsTouched();
    fixture.detectChanges();

    const error = fixture.nativeElement.querySelector('.error');
    expect(error?.textContent).toContain('Invalid email');
  });
});
```

---

## 5. Testing services

### TestBed.inject()

```typescript
// user.service.ts
@Injectable({ providedIn: 'root' })
export class UserService {
  private apiUrl = '/api/users';

  constructor(private http: HttpClient) {}

  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(this.apiUrl);
  }

  getUserById(id: number): Observable<User> {
    return this.http.get<User>(`${this.apiUrl}/${id}`);
  }

  createUser(user: Partial<User>): Observable<User> {
    return this.http.post<User>(this.apiUrl, user);
  }
}

// user.service.spec.ts
describe('UserService', () => {
  let service: UserService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [UserService]
    });

    service = TestBed.inject(UserService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => httpMock.verify());

  it('should be created', () => {
    expect(service).toBeTruthy();
  });

  it('should fetch all users', () => {
    const mockUsers: User[] = [
      { id: 1, name: 'Alice', email: 'alice@test.com' },
      { id: 2, name: 'Bob', email: 'bob@test.com' }
    ];

    service.getUsers().subscribe(users => {
      expect(users.length).toBe(2);
      expect(users).toEqual(mockUsers);
    });

    const req = httpMock.expectOne('/api/users');
    expect(req.request.method).toBe('GET');
    req.flush(mockUsers);
  });
});
```

### Testing Observables cu subscribe si marble testing

```typescript
// Varianta clasica - subscribe
it('should transform user data', (done) => {
  service.getActiveUsers().subscribe(users => {
    expect(users.every(u => u.active)).toBe(true);
    done();  // IMPORTANT: semnaleaza ca testul asincron s-a terminat
  });

  httpMock.expectOne('/api/users').flush(mockUsers);
});

// Marble testing - pentru logica complexa de Observables
import { TestScheduler } from 'rxjs/testing';

describe('DataService marble tests', () => {
  let scheduler: TestScheduler;

  beforeEach(() => {
    scheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should debounce search input', () => {
    scheduler.run(({ cold, expectObservable }) => {
      // '-a-b-c' = emit a la 10ms, b la 30ms, c la 50ms
      // Marble syntax: - = 10ms, a/b/c = valori, | = complete, # = error
      const source$ = cold('-a-b-c---|', { a: 'a', b: 'an', c: 'ang' });

      const result$ = source$.pipe(
        debounceTime(30),   // 3 frames = 30ms
        distinctUntilChanged()
      );

      // Dupa debounce(30ms), doar ultima valoare dinaintea pauzei de 30ms+
      expectObservable(result$).toBe('------c--|', { c: 'ang' });
    });
  });

  it('should retry on error then succeed', () => {
    scheduler.run(({ cold, expectObservable }) => {
      // Primul call: eroare. Al doilea: success.
      const source$ = cold('#', {}, new Error('fail'));
      const success$ = cold('--a|', { a: 'data' });

      let attempt = 0;
      const result$ = defer(() => {
        attempt++;
        return attempt === 1 ? source$ : success$;
      }).pipe(retry(1));

      expectObservable(result$).toBe('--a|', { a: 'data' });
    });
  });

  it('should merge multiple streams', () => {
    scheduler.run(({ cold, hot, expectObservable }) => {
      const a$ = cold('  -a---a---|');
      const b$ = cold('  ---b---b-|');
      const expected = ' -a-b-a-b-|';

      expectObservable(merge(a$, b$)).toBe(expected);
    });
  });
});
```

### Testing services cu dependinte

```typescript
// notification.service.ts
@Injectable({ providedIn: 'root' })
export class NotificationService {
  constructor(
    private userService: UserService,
    private emailService: EmailService,
    private logger: LoggerService
  ) {}

  async notifyUser(userId: number, message: string): Promise<boolean> {
    const user = await firstValueFrom(this.userService.getUserById(userId));
    if (!user.email) {
      this.logger.warn(`User ${userId} has no email`);
      return false;
    }
    await this.emailService.send(user.email, message);
    this.logger.info(`Notification sent to ${user.email}`);
    return true;
  }
}

// notification.service.spec.ts
describe('NotificationService', () => {
  let service: NotificationService;
  let userServiceMock: jest.Mocked<UserService>;
  let emailServiceMock: jest.Mocked<EmailService>;
  let loggerMock: jest.Mocked<LoggerService>;

  beforeEach(() => {
    userServiceMock = {
      getUserById: jest.fn()
    } as any;

    emailServiceMock = {
      send: jest.fn().mockResolvedValue(undefined)
    } as any;

    loggerMock = {
      info: jest.fn(),
      warn: jest.fn(),
      error: jest.fn()
    } as any;

    TestBed.configureTestingModule({
      providers: [
        NotificationService,
        { provide: UserService, useValue: userServiceMock },
        { provide: EmailService, useValue: emailServiceMock },
        { provide: LoggerService, useValue: loggerMock }
      ]
    });

    service = TestBed.inject(NotificationService);
  });

  it('should send notification to user with email', async () => {
    userServiceMock.getUserById.mockReturnValue(
      of({ id: 1, name: 'Alice', email: 'alice@test.com' })
    );

    const result = await service.notifyUser(1, 'Hello');

    expect(result).toBe(true);
    expect(emailServiceMock.send).toHaveBeenCalledWith('alice@test.com', 'Hello');
    expect(loggerMock.info).toHaveBeenCalled();
  });

  it('should return false if user has no email', async () => {
    userServiceMock.getUserById.mockReturnValue(
      of({ id: 2, name: 'Bob', email: '' })
    );

    const result = await service.notifyUser(2, 'Hello');

    expect(result).toBe(false);
    expect(emailServiceMock.send).not.toHaveBeenCalled();
    expect(loggerMock.warn).toHaveBeenCalledWith('User 2 has no email');
  });
});
```

---

## 6. Testing pipes si directives

### Pure pipes - instantiere directa

Pipe-urile pure sunt cele mai simple de testat: sunt functii pure, nu au dependinte.

```typescript
// truncate.pipe.ts
@Pipe({ name: 'truncate', standalone: true })
export class TruncatePipe implements PipeTransform {
  transform(value: string, maxLength: number = 50, suffix: string = '...'): string {
    if (!value) return '';
    if (value.length <= maxLength) return value;
    return value.substring(0, maxLength) + suffix;
  }
}

// truncate.pipe.spec.ts
describe('TruncatePipe', () => {
  // NU avem nevoie de TestBed pentru pure pipes
  const pipe = new TruncatePipe();

  it('should return empty string for null/undefined', () => {
    expect(pipe.transform(null as any)).toBe('');
    expect(pipe.transform(undefined as any)).toBe('');
  });

  it('should not truncate short strings', () => {
    expect(pipe.transform('hello', 10)).toBe('hello');
  });

  it('should truncate long strings with default suffix', () => {
    const input = 'a'.repeat(60);
    const result = pipe.transform(input, 50);
    expect(result.length).toBe(53);  // 50 + '...'
    expect(result.endsWith('...')).toBe(true);
  });

  it('should use custom suffix', () => {
    const result = pipe.transform('a'.repeat(60), 50, ' [more]');
    expect(result.endsWith(' [more]')).toBe(true);
  });
});
```

### Impure pipes - cu TestBed

Impure pipes pot avea dependinte (services) si trebuie testate cu TestBed.

```typescript
// translate.pipe.ts
@Pipe({ name: 'translate', standalone: true, pure: false })
export class TranslatePipe implements PipeTransform {
  constructor(private translateService: TranslateService) {}

  transform(key: string): string {
    return this.translateService.instant(key);
  }
}

// translate.pipe.spec.ts
describe('TranslatePipe', () => {
  let pipe: TranslatePipe;
  let translateService: jest.Mocked<TranslateService>;

  beforeEach(() => {
    translateService = {
      instant: jest.fn((key: string) => {
        const translations: Record<string, string> = {
          'hello': 'Salut',
          'goodbye': 'La revedere'
        };
        return translations[key] || key;
      })
    } as any;

    TestBed.configureTestingModule({
      providers: [
        TranslatePipe,
        { provide: TranslateService, useValue: translateService }
      ]
    });

    pipe = TestBed.inject(TranslatePipe);
  });

  it('should translate known keys', () => {
    expect(pipe.transform('hello')).toBe('Salut');
  });

  it('should return key for unknown translations', () => {
    expect(pipe.transform('unknown.key')).toBe('unknown.key');
  });
});
```

### Directive testing cu host component

```typescript
// highlight.directive.ts
@Directive({
  selector: '[appHighlight]',
  standalone: true
})
export class HighlightDirective {
  @Input() appHighlight = 'yellow';
  @Input() defaultColor = '';

  @HostListener('mouseenter') onMouseEnter() {
    this.highlight(this.appHighlight || this.defaultColor || 'yellow');
  }

  @HostListener('mouseleave') onMouseLeave() {
    this.highlight('');
  }

  constructor(private el: ElementRef) {}

  private highlight(color: string) {
    this.el.nativeElement.style.backgroundColor = color;
  }
}

// highlight.directive.spec.ts
@Component({
  standalone: true,
  imports: [HighlightDirective],
  template: `
    <p appHighlight="cyan" data-testid="cyan">Cyan</p>
    <p appHighlight data-testid="default">Default</p>
    <p data-testid="no-directive">No directive</p>
  `
})
class TestHostComponent {}

describe('HighlightDirective', () => {
  let fixture: ComponentFixture<TestHostComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [TestHostComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(TestHostComponent);
    fixture.detectChanges();
  });

  it('should highlight on mouseenter with specified color', () => {
    const p = fixture.debugElement.query(By.css('[data-testid="cyan"]'));
    p.triggerEventHandler('mouseenter', null);
    expect(p.nativeElement.style.backgroundColor).toBe('cyan');
  });

  it('should remove highlight on mouseleave', () => {
    const p = fixture.debugElement.query(By.css('[data-testid="cyan"]'));
    p.triggerEventHandler('mouseenter', null);
    p.triggerEventHandler('mouseleave', null);
    expect(p.nativeElement.style.backgroundColor).toBe('');
  });

  it('should use default yellow when no color specified', () => {
    const p = fixture.debugElement.query(By.css('[data-testid="default"]'));
    p.triggerEventHandler('mouseenter', null);
    expect(p.nativeElement.style.backgroundColor).toBe('yellow');
  });

  it('should not affect elements without directive', () => {
    const elements = fixture.debugElement.queryAll(By.directive(HighlightDirective));
    expect(elements.length).toBe(2);  // doar 2 elemente au directiva
  });
});
```

---

## 7. Testing formulare reactive

### Setting form values si testing validators

```typescript
// registration-form.component.ts
@Component({
  selector: 'app-registration',
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <input formControlName="username" />
      <input formControlName="email" />
      <input formControlName="password" type="password" />
      <input formControlName="confirmPassword" type="password" />
      <button type="submit" [disabled]="form.invalid">Register</button>
    </form>
  `
})
export class RegistrationFormComponent {
  form = new FormGroup({
    username: new FormControl('', [
      Validators.required,
      Validators.minLength(3),
      Validators.maxLength(20)
    ]),
    email: new FormControl('', [Validators.required, Validators.email]),
    password: new FormControl('', [
      Validators.required,
      Validators.minLength(8),
      this.passwordStrengthValidator
    ]),
    confirmPassword: new FormControl('', [Validators.required])
  }, { validators: this.passwordMatchValidator });

  submitted = new EventEmitter<any>();

  passwordStrengthValidator(control: AbstractControl): ValidationErrors | null {
    const value = control.value;
    if (!value) return null;
    const hasNumber = /\d/.test(value);
    const hasUpper = /[A-Z]/.test(value);
    if (!hasNumber || !hasUpper) {
      return { passwordStrength: true };
    }
    return null;
  }

  passwordMatchValidator(group: AbstractControl): ValidationErrors | null {
    const password = group.get('password')?.value;
    const confirm = group.get('confirmPassword')?.value;
    return password === confirm ? null : { passwordMismatch: true };
  }

  onSubmit() {
    if (this.form.valid) {
      this.submitted.emit(this.form.value);
    }
  }
}

// registration-form.component.spec.ts
describe('RegistrationFormComponent', () => {
  let component: RegistrationFormComponent;
  let fixture: ComponentFixture<RegistrationFormComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [RegistrationFormComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(RegistrationFormComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  describe('Form Validation', () => {
    it('should start as invalid', () => {
      expect(component.form.valid).toBe(false);
    });

    it('should validate username required', () => {
      const username = component.form.get('username')!;
      expect(username.errors?.['required']).toBeTruthy();

      username.setValue('abc');
      expect(username.errors).toBeNull();
    });

    it('should validate username minLength', () => {
      const username = component.form.get('username')!;
      username.setValue('ab');
      expect(username.errors?.['minlength']).toBeTruthy();
      expect(username.errors?.['minlength'].requiredLength).toBe(3);
    });

    it('should validate email format', () => {
      const email = component.form.get('email')!;
      email.setValue('not-an-email');
      expect(email.errors?.['email']).toBeTruthy();

      email.setValue('valid@test.com');
      expect(email.valid).toBe(true);
    });

    it('should validate password strength', () => {
      const password = component.form.get('password')!;
      password.setValue('weakpassword');  // no number, no uppercase
      expect(password.errors?.['passwordStrength']).toBeTruthy();

      password.setValue('Strong1password');
      expect(password.errors?.['passwordStrength']).toBeFalsy();
    });

    it('should validate password match (cross-field)', () => {
      component.form.patchValue({
        password: 'Strong1pass',
        confirmPassword: 'Different1pass'
      });
      expect(component.form.errors?.['passwordMismatch']).toBeTruthy();

      component.form.patchValue({
        confirmPassword: 'Strong1pass'
      });
      expect(component.form.errors?.['passwordMismatch']).toBeFalsy();
    });
  });

  describe('Form Submission', () => {
    const validData = {
      username: 'testuser',
      email: 'test@example.com',
      password: 'Strong1pass',
      confirmPassword: 'Strong1pass'
    };

    it('should be valid with correct data', () => {
      component.form.patchValue(validData);
      expect(component.form.valid).toBe(true);
    });

    it('should emit on valid submit', () => {
      jest.spyOn(component.submitted, 'emit');
      component.form.patchValue(validData);
      component.onSubmit();
      expect(component.submitted.emit).toHaveBeenCalledWith(validData);
    });

    it('should not emit on invalid submit', () => {
      jest.spyOn(component.submitted, 'emit');
      component.form.patchValue({ username: 'ab' });  // invalid
      component.onSubmit();
      expect(component.submitted.emit).not.toHaveBeenCalled();
    });
  });

  describe('Template Integration', () => {
    it('should disable submit button when invalid', () => {
      fixture.detectChanges();
      const btn = fixture.nativeElement.querySelector('button');
      expect(btn.disabled).toBe(true);
    });

    it('should enable submit button when valid', () => {
      component.form.patchValue({
        username: 'testuser',
        email: 'test@example.com',
        password: 'Strong1pass',
        confirmPassword: 'Strong1pass'
      });
      fixture.detectChanges();
      const btn = fixture.nativeElement.querySelector('button');
      expect(btn.disabled).toBe(false);
    });
  });
});
```

### Testing dynamic form controls (FormArray)

```typescript
// skills-form.component.ts
@Component({ /* ... */ })
export class SkillsFormComponent {
  form = new FormGroup({
    name: new FormControl('', Validators.required),
    skills: new FormArray<FormControl<string>>([])
  });

  get skills(): FormArray {
    return this.form.get('skills') as FormArray;
  }

  addSkill(skill: string = '') {
    this.skills.push(new FormControl(skill, Validators.required));
  }

  removeSkill(index: number) {
    this.skills.removeAt(index);
  }
}

// skills-form.component.spec.ts
describe('SkillsFormComponent - Dynamic Controls', () => {
  let component: SkillsFormComponent;

  beforeEach(async () => {
    // ... TestBed setup ...
    component = fixture.componentInstance;
  });

  it('should start with empty skills array', () => {
    expect(component.skills.length).toBe(0);
  });

  it('should add a skill control', () => {
    component.addSkill('Angular');
    expect(component.skills.length).toBe(1);
    expect(component.skills.at(0).value).toBe('Angular');
  });

  it('should remove a skill control', () => {
    component.addSkill('Angular');
    component.addSkill('React');
    component.removeSkill(0);
    expect(component.skills.length).toBe(1);
    expect(component.skills.at(0).value).toBe('React');
  });

  it('should validate each skill is required', () => {
    component.addSkill('');  // empty = invalid
    expect(component.skills.at(0).valid).toBe(false);

    component.skills.at(0).setValue('TypeScript');
    expect(component.skills.at(0).valid).toBe(true);
  });

  it('should invalidate form if any skill is empty', () => {
    component.form.get('name')?.setValue('John');
    component.addSkill('Angular');
    component.addSkill('');  // invalid
    expect(component.form.valid).toBe(false);
  });
});
```

---

## 8. Mocking dependencies

### Strategii de mocking in Angular

```typescript
// 1. Obiect simplu (useValue)
const mockUserService = {
  getUsers: jest.fn().mockReturnValue(of([])),
  getUserById: jest.fn().mockReturnValue(of({ id: 1, name: 'Mock' }))
};

TestBed.configureTestingModule({
  providers: [
    { provide: UserService, useValue: mockUserService }
  ]
});

// 2. jasmine.createSpyObj (Jasmine) / echivalent Jest
// Jasmine:
const spyService = jasmine.createSpyObj('UserService', ['getUsers', 'getUserById']);
spyService.getUsers.and.returnValue(of([]));

// Jest echivalent - functie helper:
function createMockService<T>(methods: (keyof T)[]): jest.Mocked<T> {
  const mock: any = {};
  methods.forEach(m => mock[m] = jest.fn());
  return mock;
}

const mockService = createMockService<UserService>(['getUsers', 'getUserById']);
mockService.getUsers.mockReturnValue(of([]));

// 3. Partial mock (useClass cu implementare partiala)
class MockUserService implements Partial<UserService> {
  getUsers() { return of([]); }
  // Nu implementam toate metodele, doar cele folosite
}

TestBed.configureTestingModule({
  providers: [
    { provide: UserService, useClass: MockUserService }
  ]
});

// 4. useFactory (pentru mock-uri cu logica)
TestBed.configureTestingModule({
  providers: [
    {
      provide: UserService,
      useFactory: () => {
        const service = createMockService<UserService>(['getUsers', 'getUserById']);
        service.getUsers.mockReturnValue(of(mockUsersData));
        return service;
      }
    }
  ]
});

// 5. Mocking InjectionToken
const API_URL = new InjectionToken<string>('API_URL');

TestBed.configureTestingModule({
  providers: [
    { provide: API_URL, useValue: 'http://test-api.com' }
  ]
});

// 6. Overriding providers in standalone components
TestBed.configureTestingModule({
  imports: [MyStandaloneComponent]
})
.overrideComponent(MyStandaloneComponent, {
  set: {
    providers: [
      { provide: UserService, useValue: mockUserService }
    ]
  }
});
```

### Pattern avansat: Auto-mock cu Proxy

```typescript
// test-utils/auto-mock.ts
export function autoMock<T>(token: Type<T>): jest.Mocked<T> {
  const proto = token.prototype;
  const methods = Object.getOwnPropertyNames(proto)
    .filter(name => name !== 'constructor' && typeof proto[name] === 'function');

  const mock: any = {};
  methods.forEach(method => {
    mock[method] = jest.fn();
  });
  return mock;
}

// Utilizare:
const mockService = autoMock(UserService);
mockService.getUsers.mockReturnValue(of([]));
```

---

## 9. Integration testing

### Testing component interactions (parent-child)

```typescript
// parent.component.ts
@Component({
  selector: 'app-parent',
  standalone: true,
  imports: [ChildComponent],
  template: `
    <h1>{{ title }}</h1>
    <app-child
      [items]="items"
      (itemSelected)="onItemSelected($event)"
    />
    <p class="selected">Selected: {{ selectedItem?.name }}</p>
  `
})
export class ParentComponent {
  title = 'Items';
  items: Item[] = [];
  selectedItem: Item | null = null;

  constructor(private itemService: ItemService) {}

  ngOnInit() {
    this.itemService.getItems().subscribe(items => this.items = items);
  }

  onItemSelected(item: Item) {
    this.selectedItem = item;
  }
}

// Integration test - testeaza parent + child impreuna
describe('ParentComponent Integration', () => {
  let fixture: ComponentFixture<ParentComponent>;
  let httpMock: HttpTestingController;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [ParentComponent],  // importa parent (include si child - standalone)
      providers: [
        ItemService,  // serviciul REAL
        provideHttpClientTesting()  // doar HTTP-ul e mock-uit
      ]
    }).compileComponents();

    fixture = TestBed.createComponent(ParentComponent);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => httpMock.verify());

  it('should load and display items in child component', () => {
    const mockItems: Item[] = [
      { id: 1, name: 'Item 1' },
      { id: 2, name: 'Item 2' }
    ];

    fixture.detectChanges();  // triggers ngOnInit

    httpMock.expectOne('/api/items').flush(mockItems);
    fixture.detectChanges();  // update view cu datele primite

    // Verifica ca child-ul a primit si randat itemele
    const childItems = fixture.debugElement.queryAll(By.css('app-child .item'));
    expect(childItems.length).toBe(2);
  });

  it('should update selected item when child emits', () => {
    fixture.detectChanges();
    httpMock.expectOne('/api/items').flush([{ id: 1, name: 'Item 1' }]);
    fixture.detectChanges();

    // Click pe item in child component
    const itemEl = fixture.debugElement.query(By.css('app-child .item'));
    itemEl.triggerEventHandler('click', null);
    fixture.detectChanges();

    const selected = fixture.nativeElement.querySelector('.selected');
    expect(selected.textContent).toContain('Item 1');
  });
});
```

### Testing router navigation

```typescript
import { RouterTestingModule } from '@angular/router/testing';
import { Router } from '@angular/router';
import { Location } from '@angular/common';

describe('App Navigation', () => {
  let router: Router;
  let location: Location;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [
        RouterTestingModule.withRoutes([
          { path: '', component: HomeComponent },
          { path: 'users', component: UserListComponent },
          { path: 'users/:id', component: UserDetailComponent },
          {
            path: 'admin',
            component: AdminComponent,
            canActivate: [AuthGuard]
          }
        ]),
        HomeComponent,
        UserListComponent,
        UserDetailComponent,
        AdminComponent
      ],
      providers: [
        { provide: AuthGuard, useValue: { canActivate: () => false } }
      ]
    }).compileComponents();

    router = TestBed.inject(Router);
    location = TestBed.inject(Location);

    // Fixture pe root component
    const fixture = TestBed.createComponent(AppComponent);
    fixture.detectChanges();

    router.initialNavigation();
  });

  it('should navigate to /users', async () => {
    await router.navigate(['/users']);
    expect(location.path()).toBe('/users');
  });

  it('should navigate to user detail', async () => {
    await router.navigate(['/users', 42]);
    expect(location.path()).toBe('/users/42');
  });

  it('should block navigation to admin when not authenticated', async () => {
    await router.navigate(['/admin']);
    expect(location.path()).not.toBe('/admin');
  });
});
```

### Testing NgRx Store integration

```typescript
import { provideMockStore, MockStore } from '@ngrx/store/testing';

describe('UserListComponent with NgRx', () => {
  let store: MockStore;
  let fixture: ComponentFixture<UserListComponent>;

  const initialState = {
    users: {
      list: [],
      loading: false,
      error: null
    }
  };

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [UserListComponent],
      providers: [
        provideMockStore({ initialState })
      ]
    }).compileComponents();

    store = TestBed.inject(MockStore);
    fixture = TestBed.createComponent(UserListComponent);
  });

  it('should display loading spinner when loading', () => {
    // Override selector
    store.overrideSelector(selectUsersLoading, true);
    store.refreshState();
    fixture.detectChanges();

    expect(fixture.nativeElement.querySelector('.spinner')).toBeTruthy();
  });

  it('should display users from store', () => {
    store.overrideSelector(selectAllUsers, [
      { id: 1, name: 'Alice' },
      { id: 2, name: 'Bob' }
    ]);
    store.refreshState();
    fixture.detectChanges();

    const items = fixture.nativeElement.querySelectorAll('.user-item');
    expect(items.length).toBe(2);
  });

  it('should dispatch loadUsers action on init', () => {
    const dispatchSpy = jest.spyOn(store, 'dispatch');
    fixture.detectChanges();  // ngOnInit

    expect(dispatchSpy).toHaveBeenCalledWith(loadUsers());
  });

  it('should display error message', () => {
    store.overrideSelector(selectUsersError, 'Failed to load users');
    store.refreshState();
    fixture.detectChanges();

    expect(fixture.nativeElement.querySelector('.error').textContent)
      .toContain('Failed to load users');
  });
});

// Testing reducers (pure functions - no TestBed needed)
describe('usersReducer', () => {
  it('should set loading on loadUsers', () => {
    const state = usersReducer(initialUsersState, loadUsers());
    expect(state.loading).toBe(true);
    expect(state.error).toBeNull();
  });

  it('should set users on loadUsersSuccess', () => {
    const users = [{ id: 1, name: 'Alice' }];
    const state = usersReducer(initialUsersState, loadUsersSuccess({ users }));
    expect(state.list).toEqual(users);
    expect(state.loading).toBe(false);
  });
});

// Testing effects
describe('UserEffects', () => {
  let effects: UserEffects;
  let actions$: Observable<Action>;
  let userService: jest.Mocked<UserService>;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        UserEffects,
        provideMockActions(() => actions$),
        { provide: UserService, useValue: { getUsers: jest.fn() } }
      ]
    });

    effects = TestBed.inject(UserEffects);
    userService = TestBed.inject(UserService) as jest.Mocked<UserService>;
  });

  it('should load users successfully', (done) => {
    const users = [{ id: 1, name: 'Alice' }];
    userService.getUsers.mockReturnValue(of(users));
    actions$ = of(loadUsers());

    effects.loadUsers$.subscribe(action => {
      expect(action).toEqual(loadUsersSuccess({ users }));
      done();
    });
  });

  it('should handle load users failure', (done) => {
    userService.getUsers.mockReturnValue(
      throwError(() => new Error('Server error'))
    );
    actions$ = of(loadUsers());

    effects.loadUsers$.subscribe(action => {
      expect(action).toEqual(loadUsersFailure({ error: 'Server error' }));
      done();
    });
  });
});
```

---

## 10. E2E testing

### Cypress basics

```bash
npm install -D cypress @cypress/schematic
ng add @cypress/schematic
```

```typescript
// cypress/e2e/login.cy.ts
describe('Login Flow', () => {
  beforeEach(() => {
    // Viziteaza pagina
    cy.visit('/login');
  });

  it('should login successfully', () => {
    // Intercepteaza API call si raspunde cu mock
    cy.intercept('POST', '/api/auth/login', {
      statusCode: 200,
      body: { token: 'fake-jwt-token', user: { name: 'Alice' } }
    }).as('loginRequest');

    // Interactioneaza cu formularul
    cy.get('[data-testid="email"]').type('alice@test.com');
    cy.get('[data-testid="password"]').type('password123');
    cy.get('[data-testid="submit"]').click();

    // Asteapta request-ul interceptat
    cy.wait('@loginRequest').its('request.body').should('deep.equal', {
      email: 'alice@test.com',
      password: 'password123'
    });

    // Verifica redirect la dashboard
    cy.url().should('include', '/dashboard');
    cy.get('[data-testid="welcome"]').should('contain', 'Alice');
  });

  it('should show error on invalid credentials', () => {
    cy.intercept('POST', '/api/auth/login', {
      statusCode: 401,
      body: { message: 'Invalid credentials' }
    });

    cy.get('[data-testid="email"]').type('wrong@test.com');
    cy.get('[data-testid="password"]').type('wrongpass');
    cy.get('[data-testid="submit"]').click();

    cy.get('[data-testid="error"]')
      .should('be.visible')
      .and('contain', 'Invalid credentials');

    // Nu ar trebui sa fi navigat
    cy.url().should('include', '/login');
  });

  it('should validate form fields', () => {
    cy.get('[data-testid="submit"]').click();
    cy.get('[data-testid="email-error"]').should('be.visible');
    cy.get('[data-testid="password-error"]').should('be.visible');
  });
});
```

### Playwright basics

```bash
npm install -D @playwright/test
npx playwright install
```

```typescript
// e2e/login.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Login Flow', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/login');
  });

  test('should login successfully', async ({ page }) => {
    // Mock API
    await page.route('**/api/auth/login', route => {
      route.fulfill({
        status: 200,
        contentType: 'application/json',
        body: JSON.stringify({ token: 'fake', user: { name: 'Alice' } })
      });
    });

    // Fill form
    await page.getByTestId('email').fill('alice@test.com');
    await page.getByTestId('password').fill('password123');
    await page.getByTestId('submit').click();

    // Verify navigation
    await expect(page).toHaveURL(/.*dashboard/);
    await expect(page.getByTestId('welcome')).toContainText('Alice');
  });

  test('should show validation errors', async ({ page }) => {
    await page.getByTestId('submit').click();

    await expect(page.getByTestId('email-error')).toBeVisible();
    await expect(page.getByTestId('password-error')).toBeVisible();
  });

  // Playwright are locator-i mai expresivi
  test('should handle complex selectors', async ({ page }) => {
    // By role
    await page.getByRole('button', { name: 'Submit' }).click();

    // By label
    await page.getByLabel('Email').fill('test@test.com');

    // By placeholder
    await page.getByPlaceholder('Enter password').fill('pass');

    // By text
    await expect(page.getByText('Welcome')).toBeVisible();

    // Chaining locators
    const form = page.locator('form.login');
    await form.getByRole('textbox').first().fill('value');
  });
});
```

### Page Object Pattern

```typescript
// cypress/support/pages/login.page.ts (Cypress)
export class LoginPage {
  visit() {
    cy.visit('/login');
  }

  getEmailInput() {
    return cy.get('[data-testid="email"]');
  }

  getPasswordInput() {
    return cy.get('[data-testid="password"]');
  }

  getSubmitButton() {
    return cy.get('[data-testid="submit"]');
  }

  getErrorMessage() {
    return cy.get('[data-testid="error"]');
  }

  login(email: string, password: string) {
    this.getEmailInput().type(email);
    this.getPasswordInput().type(password);
    this.getSubmitButton().click();
  }
}

// cypress/e2e/login.cy.ts
const loginPage = new LoginPage();

it('should login', () => {
  loginPage.visit();
  loginPage.login('alice@test.com', 'pass123');
  cy.url().should('include', '/dashboard');
});

// e2e/pages/login.page.ts (Playwright)
import { Page, Locator } from '@playwright/test';

export class LoginPage {
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorMessage: Locator;

  constructor(private page: Page) {
    this.emailInput = page.getByTestId('email');
    this.passwordInput = page.getByTestId('password');
    this.submitButton = page.getByTestId('submit');
    this.errorMessage = page.getByTestId('error');
  }

  async goto() {
    await this.page.goto('/login');
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }
}

// e2e/login.spec.ts
test('should login', async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  await loginPage.login('alice@test.com', 'pass123');
  await expect(page).toHaveURL(/.*dashboard/);
});
```

### Cypress vs Playwright - cand ce folosesti

| Criteriu | Cypress | Playwright |
|----------|---------|------------|
| **Browsere** | Chrome, Firefox, Edge (WebKit experimental) | Chrome, Firefox, Safari (WebKit nativ) |
| **Viteza** | Rapid (ruleaza in browser) | Foarte rapid (comunicare directa cu browser) |
| **Tab-uri multiple** | NU (limitare fundamentala) | DA |
| **Cross-origin** | Complicat (cy.origin()) | Nativ |
| **API Testing** | cy.request() | request context nativ |
| **Visual regression** | Plugin (percy, cypress-image-snapshot) | Built-in (toHaveScreenshot) |
| **Paralelism** | Platit (Cypress Cloud) sau nx | Gratuit, built-in |
| **Learning curve** | Usor de invatat | Moderat |
| **Debugging** | Time-travel excellent | Trace viewer excellent |
| **Community** | Mare, matur | In crestere rapida |
| **Recomandare** | Echipe mici, proiecte standard | Cross-browser critic, multi-tab, enterprise |

---

## 11. Testing best practices

### AAA Pattern (Arrange, Act, Assert)

```typescript
it('should calculate total with discount', () => {
  // ARRANGE - pregateste datele si starea
  const cart = new ShoppingCart();
  cart.addItem({ name: 'Widget', price: 100, quantity: 2 });
  const discount = 0.1;  // 10%

  // ACT - executa actiunea testata
  const total = cart.calculateTotal(discount);

  // ASSERT - verifica rezultatul
  expect(total).toBe(180);  // (100 * 2) - 10%
});

// Anti-pattern: amestecarea celor 3 faze
it('BAD: mixed arrange/act/assert', () => {
  const cart = new ShoppingCart();
  cart.addItem({ name: 'A', price: 50, quantity: 1 });
  expect(cart.itemCount).toBe(1);         // assert prea devreme
  cart.addItem({ name: 'B', price: 30, quantity: 2 });
  expect(cart.calculateTotal()).toBe(110); // alt act + assert
});
```

### Test behavior, not implementation

```typescript
// RECE (testeaza implementarea - fragil)
it('should call internal _sortItems method', () => {
  const spy = jest.spyOn(component as any, '_sortItems');
  component.loadItems();
  expect(spy).toHaveBeenCalled();  // daca redenumesti metoda, testul crapa
});

// BINE (testeaza comportamentul - stabil)
it('should display items sorted by name', () => {
  component.items = [
    { name: 'Zebra' },
    { name: 'Apple' },
    { name: 'Mango' }
  ];
  component.sortBy('name');
  fixture.detectChanges();

  const displayed = fixture.debugElement
    .queryAll(By.css('.item'))
    .map(el => el.nativeElement.textContent.trim());

  expect(displayed).toEqual(['Apple', 'Mango', 'Zebra']);
});

// RECE (testeaza metode private)
it('should set _isValid to true', () => {
  (component as any)._validate();
  expect((component as any)._isValid).toBe(true);
});

// BINE (testeaza efectul vizibil)
it('should enable submit when form is valid', () => {
  component.form.patchValue({ name: 'Alice', email: 'alice@test.com' });
  fixture.detectChanges();
  expect(fixture.nativeElement.querySelector('button').disabled).toBe(false);
});
```

### Teste independente

```typescript
// RECE - testele depind unul de altul
describe('UserService', () => {
  let createdUserId: number;

  it('should create user', () => {
    createdUserId = service.create({ name: 'Alice' }).id;
    expect(createdUserId).toBeDefined();
  });

  it('should fetch the created user', () => {
    // DEPINDE de testul anterior - daca ala esueaza, si asta esueaza
    const user = service.getById(createdUserId);
    expect(user.name).toBe('Alice');
  });
});

// BINE - fiecare test e independent
describe('UserService', () => {
  it('should create user', () => {
    const user = service.create({ name: 'Alice' });
    expect(user.id).toBeDefined();
  });

  it('should fetch user by id', () => {
    // Arrange: creaza propriul user
    const created = service.create({ name: 'Bob' });

    // Act
    const fetched = service.getById(created.id);

    // Assert
    expect(fetched.name).toBe('Bob');
  });
});
```

### Sumar best practices

1. **AAA pattern** - Separa clar Arrange, Act, Assert
2. **Un concept per test** - Ideal o singura asertiune (sau asertiuni inrudite)
3. **Testeaza comportament, nu implementare** - Nu testa metode private
4. **Teste independente** - Nu depinde de ordinea de executie
5. **Naming descriptiv** - `should show error when email is invalid` nu `test1`
6. **DRY in teste (cu masura)** - `beforeEach` pentru setup comun, dar nu abstractiza prea mult
7. **Teste rapide** - Daca un test dureaza > 1s, e probabil un integration test
8. **data-testid** - Foloseste atribute dedicate, nu clase CSS sau structura DOM
9. **Nu testa framework-ul** - Nu testa ca `*ngIf` functioneaza
10. **Red-Green-Refactor** - Scrie testul inainte de cod (TDD)

---

## 12. Code coverage si metrici

### Istanbul coverage reports

```typescript
// jest.config.ts
const config: Config = {
  collectCoverage: true,
  coverageDirectory: 'coverage',
  coverageReporters: ['html', 'text', 'text-summary', 'lcov'],
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.spec.ts',
    '!src/**/*.module.ts',
    '!src/main.ts',
    '!src/**/index.ts',       // barrel files
    '!src/**/*.interface.ts',
    '!src/**/*.model.ts',
    '!src/environments/**'
  ]
};
```

### Coverage thresholds

```typescript
// jest.config.ts
const config: Config = {
  coverageThreshold: {
    // Global thresholds
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    },
    // Per-folder thresholds (mai stricte pentru business logic)
    './src/app/core/services/': {
      branches: 90,
      functions: 95,
      lines: 90,
      statements: 90
    },
    // Mai permisive pentru componente UI simple
    './src/app/shared/components/': {
      branches: 60,
      functions: 70,
      lines: 70,
      statements: 70
    }
  }
};
```

### Tipuri de coverage si importanta lor

```
| Tip                | Ce masoara                           | Importanta |
|--------------------|--------------------------------------|------------|
| Statement coverage | % din instructiuni executate         | Baza       |
| Branch coverage    | % din ramuri if/else/switch acoperite| CRITICA    |
| Function coverage  | % din functii apelate                | Moderata   |
| Line coverage      | % din linii executate                | Baza       |
```

**Branch coverage este CEL MAI IMPORTANT** - acopera edge cases:

```typescript
// Exemplu: 100% line coverage dar 50% branch coverage
function getDiscount(user: User): number {
  if (user.isPremium && user.yearsActive > 5) {
    return 0.2;  // 20%
  }
  return 0;
}

// Test cu 100% line coverage dar doar 50% branch coverage:
it('should return 0 for non-premium', () => {
  expect(getDiscount({ isPremium: false, yearsActive: 1 })).toBe(0);
});
it('should return 20% for premium + 5 years', () => {
  expect(getDiscount({ isPremium: true, yearsActive: 6 })).toBe(0.2);
});
// LIPSESTE: premium cu < 5 ani (branch: isPremium=true && yearsActive<=5)
// LIPSESTE: non-premium cu > 5 ani
```

### Meaningful coverage vs 100% coverage

**100% coverage NU inseamna 0 bug-uri.** Coverage masoara ce cod a fost executat, NU ce a fost verificat corect.

```typescript
// Test cu 100% coverage dar ZERO valoare
it('should not throw', () => {
  expect(() => service.processOrder(mockOrder)).not.toThrow();
  // Executa tot codul dar nu verifica NIMIC despre rezultat
});

// Test cu mai putin coverage dar valoare reala
it('should calculate order total with tax and shipping', () => {
  const order = createOrder([
    { price: 100, quantity: 2 },
    { price: 50, quantity: 1 }
  ]);

  const result = service.processOrder(order);

  expect(result.subtotal).toBe(250);
  expect(result.tax).toBe(47.5);        // 19% TVA
  expect(result.shipping).toBe(0);       // free over 200
  expect(result.total).toBe(297.5);
});
```

**Praguri realiste recomandate:**

| Zona | Coverage target | Motivatie |
|------|----------------|-----------|
| Core business logic | 90%+ | Logica critica, bugurile costa mult |
| Services / API layer | 85%+ | Integrari importante |
| Components (logica) | 75-85% | Template-uri simple nu au nevoie de 100% |
| Shared/UI components | 60-70% | Multe sunt wrappere simple |
| Models/Interfaces | N/A | Nu au logica de testat |
| Config/Bootstrap | Exclus | `main.ts`, `app.config.ts` |

---

## Intrebari frecvente de interviu

### 1. Care e diferenta intre `fakeAsync/tick` si `waitForAsync`? Cand folosesti fiecare?

**Raspuns:** `fakeAsync` simuleaza trecerea timpului in mod sincron. Cu `tick(ms)` avansezi manual ceasul virtual, iar cu `flush()` termini toate operatiile pending. E preferat pentru ca testele raman sincrone si predictibile.

`waitForAsync` (fost `async()`) foloseste Zone.js real si asteapta toate Promises sa se rezolve. E util cand ai Promises in lant sau `async/await` care nu pot fi simulate cu `fakeAsync`. E mai rar folosit.

Regula: **foloseste `fakeAsync` by default**, `waitForAsync` doar cand `fakeAsync` nu functioneaza (ex: XHR real, macrotask-uri complexe).

### 2. Ce e shallow testing si cand il folosesti?

**Raspuns:** Shallow testing inseamna testarea componentei fara a randa componentele copil. Se face cu `NO_ERRORS_SCHEMA` sau `CUSTOM_ELEMENTS_SCHEMA` in TestBed.

Avantaje: teste rapide, izolate, nu depind de implementarea child-urilor.
Dezavantaje: nu detecteaza erori de binding (un @Input redenumit in child nu va fi detectat).

Il folosesti cand: vrei sa testezi doar logica componentei parinte, sau cand child-urile sunt complexe si ar incetini testele. Pentru testare completa, combina shallow tests cu cateva integration tests.

### 3. Cum testezi un guard care depinde de un service asincron?

**Raspuns:**

```typescript
describe('AuthGuard', () => {
  let guard: AuthGuard;
  let authService: jest.Mocked<AuthService>;
  let router: Router;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [RouterTestingModule],
      providers: [
        AuthGuard,
        {
          provide: AuthService,
          useValue: { isAuthenticated: jest.fn() }
        }
      ]
    });

    guard = TestBed.inject(AuthGuard);
    authService = TestBed.inject(AuthService) as jest.Mocked<AuthService>;
    router = TestBed.inject(Router);
  });

  it('should allow access when authenticated', (done) => {
    authService.isAuthenticated.mockReturnValue(of(true));

    const result$ = guard.canActivate(
      {} as ActivatedRouteSnapshot,
      { url: '/admin' } as RouterStateSnapshot
    );

    (result$ as Observable<boolean>).subscribe(allowed => {
      expect(allowed).toBe(true);
      done();
    });
  });

  it('should redirect to login when not authenticated', (done) => {
    authService.isAuthenticated.mockReturnValue(of(false));
    const navigateSpy = jest.spyOn(router, 'navigate');

    const result$ = guard.canActivate(
      {} as ActivatedRouteSnapshot,
      { url: '/admin' } as RouterStateSnapshot
    );

    (result$ as Observable<boolean>).subscribe(allowed => {
      expect(allowed).toBe(false);
      expect(navigateSpy).toHaveBeenCalledWith(['/login']);
      done();
    });
  });
});
```

### 4. Cum se testeaza un interceptor HTTP?

**Raspuns:**

```typescript
describe('AuthInterceptor', () => {
  let httpMock: HttpTestingController;
  let http: HttpClient;
  let tokenService: jest.Mocked<TokenService>;

  beforeEach(() => {
    tokenService = { getToken: jest.fn() } as any;

    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [
        { provide: TokenService, useValue: tokenService },
        {
          provide: HTTP_INTERCEPTORS,
          useClass: AuthInterceptor,
          multi: true
        }
      ]
    });

    http = TestBed.inject(HttpClient);
    httpMock = TestBed.inject(HttpTestingController);
  });

  it('should add Authorization header', () => {
    tokenService.getToken.mockReturnValue('my-token');

    http.get('/api/data').subscribe();

    const req = httpMock.expectOne('/api/data');
    expect(req.request.headers.get('Authorization'))
      .toBe('Bearer my-token');
  });

  it('should not add header when no token', () => {
    tokenService.getToken.mockReturnValue(null);

    http.get('/api/data').subscribe();

    const req = httpMock.expectOne('/api/data');
    expect(req.request.headers.has('Authorization')).toBe(false);
  });
});
```

### 5. Care e diferenta intre `toBe` si `toEqual`?

**Raspuns:** `toBe` foloseste `===` (strict equality / referinta identica). `toEqual` face deep equality (compara recursiv proprietatile obiectelor).

```typescript
const a = { x: 1 };
const b = { x: 1 };

expect(a).toBe(b);      // FAIL - obiecte diferite in memorie
expect(a).toEqual(b);    // PASS - aceleasi proprietati
expect(a).toBe(a);       // PASS - aceeasi referinta

// Regula: toBe pentru primitive, toEqual pentru obiecte si arrays
expect(5).toBe(5);                   // OK
expect([1, 2]).toEqual([1, 2]);      // OK
expect({ a: 1 }).toEqual({ a: 1 }); // OK
```

### 6. Cum testezi un component care foloseste ActivatedRoute?

**Raspuns:**

```typescript
describe('UserDetailComponent', () => {
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [UserDetailComponent],
      providers: [
        {
          provide: ActivatedRoute,
          useValue: {
            params: of({ id: '42' }),
            queryParams: of({ tab: 'profile' }),
            snapshot: {
              paramMap: convertToParamMap({ id: '42' })
            }
          }
        },
        { provide: UserService, useValue: mockUserService }
      ]
    }).compileComponents();
  });

  it('should load user from route param', () => {
    mockUserService.getUserById.mockReturnValue(
      of({ id: 42, name: 'Alice' })
    );
    fixture.detectChanges();

    expect(mockUserService.getUserById).toHaveBeenCalledWith(42);
    expect(component.user?.name).toBe('Alice');
  });
});
```

### 7. Ce sunt Component Harnesses si de ce le-ai folosi?

**Raspuns:** Component Harnesses (din `@angular/cdk/testing`) sunt o abstractizare peste DOM care ofera un API stabil pentru interactiunea cu componentele in teste. Functia lor principala: decupleaza testele de structura DOM.

Avantaje:
- **Rezistente la refactoring**: daca schimbi template-ul (CSS class, structura), harness-ul absoarbe schimbarea
- **API asincron consistent**: toate operatiile sunt `async`, gestioneaza automat change detection
- **Reutilizabile**: acelasi harness merge in unit tests si E2E (Protractor/Playwright)
- **Encapsulare**: consumatorul nu trebuie sa stie structura DOM interna a componentei

Angular Material ofera harnesses pentru toate componentele (`MatButtonHarness`, `MatInputHarness`, etc.). Pentru componentele proprii, creezi harnesses custom extinzand `ComponentHarness`.

### 8. Cum abordezi testarea unei aplicatii Angular existente care nu are teste?

**Raspuns:** Ca Principal Engineer, strategia e graduala:

1. **Configureaza infrastructura**: Jest, coverage reports, CI integration
2. **Stabileste coverage baseline**: ruleaza coverage si documenteaza starea curenta (probabil < 10%)
3. **Regula "boy scout"**: orice cod modificat trebuie sa aiba teste (nu rescrii tot)
4. **Prioritizeaza**: incepe cu business logic critica (services cu logica de business, validators, pipes)
5. **Integration tests**: adauga teste pentru critical user journeys (login, checkout)
6. **Coverage gates in CI**: seteaza threshold la baseline + 5%, creste gradual
7. **E2E smoke tests**: 5-10 teste pentru cele mai importante flows
8. **Nu testa retroactiv cod legacy stabil**: daca nu se modifica, nu adauga teste doar pentru coverage

Obiectivul: de la 0% la 70% coverage in 3-6 luni, cu focus pe cod nou si cod modificat frecvent.

### 9. Cand folosesti Cypress si cand Playwright pentru E2E?

**Raspuns:** **Cypress** e ideal pentru echipe mici/medii, proiecte cu un singur domain, cand vrei time-travel debugging si o experienta excelenta de developer. Limitari: nu suporta multi-tab, cross-origin e complicat, paralelismul e platit.

**Playwright** e ideal pentru: teste cross-browser (inclusiv Safari/WebKit), scenarii multi-tab sau multi-origin, proiecte enterprise cu nevoie de paralelism gratuit, si cand ai nevoie de visual regression testing built-in. Are si API testing nativ superior.

Tendinta industriei: Playwright castiga teren rapid datorita vitezei, feature-urilor si faptului ca e complet gratuit (inclusiv paralelism).

### 10. Explica marble testing si cand e util.

**Raspuns:** Marble testing e o tehnica de testare a Observables folosind o sintaxa vizuala (marble diagrams). Foloseste `TestScheduler` din RxJS.

Sintaxa: `-` = 1 frame (10ms virtual), `a`/`b`/`c` = valori emise, `|` = complete, `#` = error, `^` = subscription point, `(abc)` = valori sincrone grupate.

E util cand testezi: operatori de timp (`debounceTime`, `delay`, `throttle`), combinari de streams (`merge`, `combineLatest`, `switchMap`), retry logic, si orice logica RxJS complexa.

Nu e necesar pentru: Observables simple (`of()`, `from()`), HTTP calls (foloseste `HttpTestingController`), sau cand `subscribe` + `done` e suficient. Marble testing adauga complexitate si trebuie folosit doar cand ofera valoare reala.
