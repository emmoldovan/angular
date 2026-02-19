# Angular Core - Concepte Avansate (Interview Prep - Principal Software Engineer)

> Acest document acopera fundamentele avansate ale Angular (v17-19), cu accent pe
> standalone components, lifecycle hooks, content projection, signal queries,
> noul control flow si compilatorul Angular.

---

## 1. Standalone Components

### Ce sunt Standalone Components?

Standalone components sunt componente Angular care **nu depind de un `NgModule`** pentru a-si declara dependentele. Începând cu **Angular 19** standalone este implicit; din **Angular 20+** Style Guide recomandă **să nu mai setezi** `standalone: true` în decorator (e default).

Inainte de standalone, fiecare componenta trebuia declarata intr-un `NgModule`, iar dependentele (pipe-uri, directive, alte componente) erau importate la nivel de modul. Acest lucru ducea la "NgModule bloat" - module mari, greu de intretinut, cu dependente neclare.

### Diferente fata de componente bazate pe NgModule

| Aspect | NgModule-based | Standalone |
|--------|---------------|------------|
| Declarare dependente | In `NgModule.imports/declarations` | Direct in `@Component({ imports: [...] })` |
| Tree-shaking | Slab - modulul intreg este inclus | Excelent - doar ce e folosit |
| Lazy loading | La nivel de modul (`loadChildren`) | La nivel de componenta (`loadComponent`) |
| Boilerplate | Ridicat (module + componente) | Minimal (doar componenta) |
| Default din Angular 19 | Nu | Da |

### Exemplu practic

```typescript
// Angular 19 - standalone este implicit, nu mai trebuie specificat
@Component({
  selector: 'app-user-profile',
  // In v20+ do NOT set standalone: true — it's the default (angular.dev style guide)
  imports: [CommonModule, ReactiveFormsModule, UserAvatarComponent],
  template: `
    <div class="profile">
      <app-user-avatar [user]="user()" />
      @if (isEditing()) {
        <form [formGroup]="profileForm">
          <input formControlName="name" />
          <input formControlName="email" />
        </form>
      } @else {
        <p>{{ user().name }}</p>
        <p>{{ user().email }}</p>
      }
    </div>
  `
})
export class UserProfileComponent {
  user = input.required<User>();
  isEditing = signal(false);
  profileForm = inject(FormBuilder).group({
    name: [''],
    email: ['']
  });
}
```

### Bootstrap fara NgModule

```typescript
// main.ts - bootstrapApplication inlocuieste platformBrowserDynamic().bootstrapModule()
import { bootstrapApplication } from '@angular/platform-browser';
import { provideRouter } from '@angular/router';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { AppComponent } from './app/app.component';
import { routes } from './app/app.routes';
import { authInterceptor } from './app/interceptors/auth.interceptor';

bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes),
    provideHttpClient(withInterceptors([authInterceptor])),
    provideAnimationsAsync(),
  ]
});
```

### Lazy loading la nivel de componenta

```typescript
// app.routes.ts
export const routes: Routes = [
  {
    path: 'dashboard',
    loadComponent: () => import('./dashboard/dashboard.component')
      .then(m => m.DashboardComponent)
  },
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.routes')
      .then(m => m.ADMIN_ROUTES)
  }
];
```

### Migrarea de la NgModule la Standalone

Angular CLI ofera un schematic dedicat:

```bash
# Pasul 1: Converteste componentele la standalone
ng generate @angular/core:standalone

# Schematic-ul face 3 pasi (poate fi rulat pas cu pas):
# --mode=convert-to-standalone   -> adauga standalone: true, muta imports
# --mode=prune-ng-modules        -> sterge modulele devenite inutile
# --mode=standalone-bootstrap    -> converteste bootstrap-ul aplicatiei
```

**Strategia recomandata pentru migrare graduala:**

1. Incepe cu componentele **leaf** (fara copii) - cele mai simple
2. Foloseste `importProvidersFrom()` pentru a reutiliza provideri din module existente
3. Converteste rutele la `loadComponent` pe masura ce avansezi
4. La final, elimina `AppModule` si treci la `bootstrapApplication`

```typescript
// Strategie hibrida: componenta standalone care foloseste un modul existent
@Component({
  selector: 'app-hybrid',
  imports: [SharedModule], // poti importa un NgModule intreg intr-un standalone component
  template: `<shared-button (click)="doSomething()">Click</shared-button>`
})
export class HybridComponent {}

// In providers, poti reutiliza provideri din module
bootstrapApplication(AppComponent, {
  providers: [
    importProvidersFrom(ExistingCoreModule), // reutilizeaza provideri din modul
    provideRouter(routes),
  ]
});
```

---

## 2. Lifecycle Hooks

### Ordinea completa de executie

Lifecycle hooks sunt metode predefinite pe care Angular le apeleaza in momente specifice din viata unei componente sau directive. Ordinea este **stricta si determinista**:

```
1. constructor()              -> instantierea clasei (NU e hook Angular)
2. ngOnChanges()              -> cand @Input()-urile se schimba (inainte de ngOnInit)
3. ngOnInit()                 -> dupa prima initializare a input-urilor
4. ngDoCheck()                -> la fiecare ciclu change detection
5. ngAfterContentInit()       -> dupa prima proiectie a continutului (ng-content)
6. ngAfterContentChecked()    -> dupa fiecare verificare a continutului proiectat
7. ngAfterViewInit()          -> dupa prima initializare a view-ului (copii din template)
8. ngAfterViewChecked()       -> dupa fiecare verificare a view-ului
9. ngOnDestroy()              -> inainte de distrugerea componentei
```

### Diagrama vizuala a ciclului de viata

```
Creare componenta
       |
  constructor()
       |
  ngOnChanges()  <--- se apeleaza si la fiecare schimbare ulterioara de @Input()
       |
  ngOnInit()     <--- o singura data
       |
  ngDoCheck()    <-------+
       |                  |
  ngAfterContentInit()    |  (o singura data)
       |                  |
  ngAfterContentChecked() |
       |                  |
  ngAfterViewInit()       |  (o singura data)
       |                  |
  ngAfterViewChecked() ---+  (ciclul se repeta la fiecare change detection)
       |
  ngOnDestroy()  <--- la distrugere
```

### Detalii si use-cases pentru fiecare hook

```typescript
@Component({
  selector: 'app-lifecycle-demo',
  template: `
    <ng-content />
    <app-child [data]="data()" />
  `
})
export class LifecycleDemoComponent implements
    OnChanges, OnInit, DoCheck, AfterContentInit,
    AfterContentChecked, AfterViewInit, AfterViewChecked, OnDestroy {

  data = input<string>();

  private destroyRef = inject(DestroyRef);
  private logger = inject(LoggerService);

  // 1. CONSTRUCTOR
  // NU este un hook Angular, ci constructorul TypeScript.
  // Se executa inainte ca Angular sa initializeze input-urile.
  // Use-case: injectare dependente (prefer inject() in Angular 14+)
  // EVITA: logica complexa, apeluri HTTP, acces la DOM
  constructor() {
    // In Angular modern, prefer inject() la nivel de camp:
    // private http = inject(HttpClient);
  }

  // 2. ngOnChanges
  // Se apeleaza INAINTE de ngOnInit si la FIECARE schimbare de @Input()
  // Primeste un SimpleChanges object cu valorile anterioare si curente
  // Use-case: reactie la schimbari de input, validare input-uri
  ngOnChanges(changes: SimpleChanges): void {
    if (changes['data']) {
      const prev = changes['data'].previousValue;
      const curr = changes['data'].currentValue;
      const isFirst = changes['data'].firstChange;
      this.logger.log(`data: ${prev} -> ${curr} (first: ${isFirst})`);
    }
  }

  // 3. ngOnInit
  // Se apeleaza O SINGURA DATA, dupa ce input-urile sunt initializate
  // Use-case: initializare logica, fetch date, setup subscriptions
  // IMPORTANT: input-urile SUNT disponibile aici (spre deosebire de constructor)
  ngOnInit(): void {
    this.loadData(this.data());
    // Alternativa moderna: DestroyRef + takeUntilDestroyed
  }

  // 4. ngDoCheck
  // Se apeleaza la FIECARE ciclu de change detection
  // Use-case: detectare manuala a schimbarilor pe care Angular nu le vede
  //           (ex: mutarea unui obiect, schimbari in colectii)
  // ATENTIE: se apeleaza foarte frecvent - trebuie sa fie EXTREM de rapid
  ngDoCheck(): void {
    // Verifica schimbari pe care Angular nu le detecteaza automat
    // Ex: deep comparison pe obiecte complexe
  }

  // 5. ngAfterContentInit
  // Se apeleaza O SINGURA DATA, dupa ce Angular proiecteaza continutul
  // extern (ng-content) in componenta
  // Use-case: acces la @ContentChild/@ContentChildren
  ngAfterContentInit(): void {
    // ContentChild-urile sunt disponibile AICI, nu in ngOnInit
  }

  // 6. ngAfterContentChecked
  // Se apeleaza dupa FIECARE verificare a continutului proiectat
  // Use-case: reactii la schimbari in continutul proiectat
  // ATENTIE: se apeleaza frecvent - trebuie sa fie rapid
  ngAfterContentChecked(): void {}

  // 7. ngAfterViewInit
  // Se apeleaza O SINGURA DATA, dupa ce view-ul componentei si
  // copiii sai sunt complet initializati
  // Use-case: acces la @ViewChild/@ViewChildren, initializare librarii DOM
  //           (charturi, editoare, etc.)
  ngAfterViewInit(): void {
    // ViewChild-urile sunt disponibile AICI
    // Poti accesa elementele DOM in siguranta
  }

  // 8. ngAfterViewChecked
  // Se apeleaza dupa FIECARE verificare a view-ului componentei
  // Use-case: reactii la schimbari in view
  // ATENTIE: NU modifica starea componentei aici -> cauzeaza
  //          ExpressionChangedAfterItHasBeenCheckedError
  ngAfterViewChecked(): void {}

  // 9. ngOnDestroy
  // Se apeleaza inainte ca Angular sa distruga componenta
  // Use-case: cleanup - unsubscribe, clearInterval, detach event listeners
  ngOnDestroy(): void {
    this.logger.log('Componenta distrusa');
    // In Angular modern, prefer DestroyRef:
  }
}
```

### Alternativa moderna: DestroyRef si afterNextRender

```typescript
@Component({ /* ... */ })
export class ModernLifecycleComponent {
  private destroyRef = inject(DestroyRef);
  private http = inject(HttpClient);

  data = input.required<string>();

  // Efect care ruleaza la fiecare schimbare de input (inlocuieste ngOnChanges)
  dataEffect = effect(() => {
    const currentData = this.data();
    console.log('Data s-a schimbat:', currentData);
  });

  constructor() {
    // Inlocuieste ngOnDestroy - mai sigur, functioneaza si in directive
    this.destroyRef.onDestroy(() => {
      console.log('Cleanup executat');
    });

    // Inlocuieste ngAfterViewInit pentru operatii DOM (SSR-safe)
    afterNextRender(() => {
      // Acces sigur la DOM, ruleaza doar in browser
      this.initializeChart();
    });

    // afterRender ruleaza dupa FIECARE renderizare
    afterRender(() => {
      // Echivalent cu ngAfterViewChecked, dar SSR-aware
    });
  }

  private initializeChart(): void { /* ... */ }
}
```

---

## 3. Content Projection

Content projection permite unei componente sa accepte continut din exterior si sa-l afiseze in template-ul propriu. Este echivalentul Angular pentru conceptul de "slots" din Web Components sau "children" din React.

### 3.1 Single-slot projection (ng-content)

```typescript
// card.component.ts
@Component({
  selector: 'app-card',
  template: `
    <div class="card">
      <div class="card-body">
        <ng-content />  <!-- tot continutul din exterior vine aici -->
      </div>
    </div>
  `
})
export class CardComponent {}

// Utilizare:
// <app-card>
//   <h2>Titlu</h2>
//   <p>Continut proiectat in card</p>
// </app-card>
```

### 3.2 Multi-slot projection (ng-content cu select)

```typescript
// panel.component.ts
@Component({
  selector: 'app-panel',
  template: `
    <div class="panel">
      <header class="panel-header">
        <ng-content select="[panel-header]" />
      </header>

      <div class="panel-body">
        <ng-content />  <!-- default slot: prinde tot ce nu e selectat -->
      </div>

      <footer class="panel-footer">
        <ng-content select="[panel-footer]" />
      </footer>
    </div>
  `
})
export class PanelComponent {}

// Utilizare:
// <app-panel>
//   <div panel-header>Titlul Panelului</div>
//   <p>Acesta este continutul principal (merge in slot-ul default)</p>
//   <span>Si acesta merge in slot-ul default</span>
//   <div panel-footer>Footer cu butoane</div>
// </app-panel>
```

**Tipuri de selectori pentru `ng-content select`:**

```html
<!-- Selector pe element -->
<ng-content select="header" />

<!-- Selector pe atribut -->
<ng-content select="[slot-name]" />

<!-- Selector pe clasa CSS -->
<ng-content select=".highlighted" />

<!-- Selector pe directiva -->
<ng-content select="appHighlight" />

<!-- Combinatii -->
<ng-content select="div[role='alert']" />
```

### 3.3 ng-template si ngTemplateOutlet

`ng-template` defineste un template **care nu se randeaza automat**. Poate fi randat programatic cu `ngTemplateOutlet` sau prin directive structurale.

```typescript
@Component({
  selector: 'app-data-list',
  imports: [NgTemplateOutlet],
  template: `
    <div class="list">
      @for (item of items(); track item.id) {
        <!-- Foloseste template-ul custom daca exista, altfel cel default -->
        <ng-container
          [ngTemplateOutlet]="itemTemplate() || defaultTemplate"
          [ngTemplateOutletContext]="{ $implicit: item, index: $index }"
        />
      }
    </div>

    <!-- Template default, folosit cand nu se furnizeaza unul custom -->
    <ng-template #defaultTemplate let-item let-i="index">
      <div class="default-item">
        {{ i + 1 }}. {{ item.name }}
      </div>
    </ng-template>
  `
})
export class DataListComponent<T> {
  items = input.required<T[]>();
  itemTemplate = contentChild<TemplateRef<any>>('itemTemplate');
}

// Utilizare cu template custom:
// <app-data-list [items]="users()">
//   <ng-template #itemTemplate let-user let-idx="index">
//     <div class="custom-item">
//       <img [src]="user.avatar" />
//       <span>{{ user.name }}</span>
//       <small>Pozitia: {{ idx }}</small>
//     </div>
//   </ng-template>
// </app-data-list>
```

### 3.4 ng-container

`ng-container` este un element logic Angular care **nu produce niciun element DOM**. Este util pentru:

1. **Gruparea elementelor fara markup extra:**

```html
<!-- GRESIT: div-ul extra strica layout-ul CSS -->
<div *ngIf="showItems">
  <li>Item 1</li>
  <li>Item 2</li>
</div>

<!-- CORECT: ng-container nu adauga niciun element DOM -->
<ng-container *ngIf="showItems">
  <li>Item 1</li>
  <li>Item 2</li>
</ng-container>

<!-- Angular 17+ cu noul control flow, ng-container nu mai e necesar: -->
@if (showItems) {
  <li>Item 1</li>
  <li>Item 2</li>
}
```

2. **Aplicarea a doua directive structurale** (nu poti pune doua pe acelasi element):

```html
<!-- EROARE: nu poti avea *ngFor si *ngIf pe acelasi element -->
<li *ngFor="let item of items" *ngIf="item.active">{{ item.name }}</li>

<!-- CORECT cu ng-container: -->
<ng-container *ngFor="let item of items">
  <li *ngIf="item.active">{{ item.name }}</li>
</ng-container>

<!-- Angular 17+ - noul control flow rezolva problema nativ: -->
@for (item of items; track item.id) {
  @if (item.active) {
    <li>{{ item.name }}</li>
  }
}
```

3. **Cu ngTemplateOutlet** (vezi exemplul de mai sus)

### 3.5 Directive structurale custom

Directivele structurale manipuleaza DOM-ul prin adaugarea sau stergerea elementelor. Prefixul `*` este syntactic sugar pentru `<ng-template>`.

```typescript
// unless.directive.ts - opusul lui *ngIf
@Directive({
  selector: '[appUnless]'
})
export class UnlessDirective {
  private templateRef = inject(TemplateRef<any>);
  private viewContainer = inject(ViewContainerRef);
  private hasView = false;

  @Input()
  set appUnless(condition: boolean) {
    if (!condition && !this.hasView) {
      this.viewContainer.createEmbeddedView(this.templateRef);
      this.hasView = true;
    } else if (condition && this.hasView) {
      this.viewContainer.clear();
      this.hasView = false;
    }
  }
}

// Utilizare:
// <div *appUnless="isLoading">Continutul s-a incarcat!</div>
//
// Angular expandeaza *appUnless in:
// <ng-template [appUnless]="isLoading">
//   <div>Continutul s-a incarcat!</div>
// </ng-template>
```

---

## 4. ViewChild, ContentChild, ViewChildren, ContentChildren

### Diferenta fundamentala: View vs Content

- **View children** = elemente din **template-ul componentei** (definite in `template` / `templateUrl`)
- **Content children** = elemente **proiectate din exterior** (prin `<ng-content>`)

```typescript
@Component({
  selector: 'app-parent',
  template: `
    <!-- Acestia sunt VIEW CHILDREN (definiti in template-ul componentei) -->
    <div #myDiv>View child element</div>
    <app-child-a />

    <!-- Continutul proiectat din exterior va fi CONTENT CHILDREN -->
    <ng-content />
  `
})
export class ParentComponent {}

// Cand folosesti ParentComponent:
// <app-parent>
//   <!-- Acestia sunt CONTENT CHILDREN (proiectati din exterior) -->
//   <app-child-b />
//   <p #projected>Continut proiectat</p>
// </app-parent>
```

### 4.1 API-ul clasic (decorator-based)

```typescript
@Component({
  selector: 'app-classic-queries',
  template: `
    <input #searchInput type="text" />
    <app-item *ngFor="let item of items" [data]="item" />
    <ng-content />
  `
})
export class ClassicQueriesComponent implements AfterViewInit, AfterContentInit {
  // ViewChild - un singur element din view
  @ViewChild('searchInput') searchInput!: ElementRef<HTMLInputElement>;
  @ViewChild(ItemComponent) firstItem!: ItemComponent;

  // ViewChildren - toate elementele de un tip din view
  @ViewChildren(ItemComponent) allItems!: QueryList<ItemComponent>;

  // ContentChild - un singur element proiectat
  @ContentChild('projected') projectedElement!: ElementRef;
  @ContentChild(ChildBComponent) contentChild!: ChildBComponent;

  // ContentChildren - toate elementele proiectate de un tip
  @ContentChildren(ChildBComponent) contentChildren!: QueryList<ChildBComponent>;

  // ViewChild-urile sunt disponibile in ngAfterViewInit
  ngAfterViewInit(): void {
    this.searchInput.nativeElement.focus();
    console.log(`Total items in view: ${this.allItems.length}`);

    // QueryList este observabil - reactioneaza la schimbari
    this.allItems.changes.subscribe((items: QueryList<ItemComponent>) => {
      console.log(`Items changed: ${items.length}`);
    });
  }

  // ContentChild-urile sunt disponibile in ngAfterContentInit
  ngAfterContentInit(): void {
    console.log('Projected element:', this.projectedElement);
    console.log(`Content children: ${this.contentChildren.length}`);
  }
}
```

### 4.2 Signal-based queries (Angular 17+)

Incepand cu Angular 17, queries pot fi definite ca **signal-uri**. Aceasta este abordarea recomandata in Angular modern.

```typescript
@Component({
  selector: 'app-signal-queries',
  template: `
    <input #searchInput type="text" />
    <div #container class="item-container">
      @for (item of items(); track item.id) {
        <app-item [data]="item" />
      }
    </div>
    <ng-content />
  `
})
export class SignalQueriesComponent {
  items = input.required<Item[]>();

  // viewChild() - returneaza Signal<ElementRef | undefined>
  searchInput = viewChild<ElementRef<HTMLInputElement>>('searchInput');

  // viewChild.required() - garanteaza ca elementul exista
  container = viewChild.required<ElementRef>('container');

  // viewChildren() - returneaza Signal<ReadonlyArray<ItemComponent>>
  allItems = viewChildren(ItemComponent);

  // contentChild() - element proiectat din exterior
  projectedHeader = contentChild<ElementRef>('header');

  // contentChildren() - toate elementele proiectate de un tip
  projectedItems = contentChildren(ItemComponent);

  constructor() {
    // Reactie automata la schimbari prin effect()
    effect(() => {
      const input = this.searchInput();
      if (input) {
        input.nativeElement.focus();
      }
    });

    // Signal-urile se actualizeaza automat cand copiii se schimba
    effect(() => {
      const items = this.allItems();
      console.log(`View contine ${items.length} item-uri`);
    });

    effect(() => {
      const projected = this.projectedItems();
      console.log(`Content contine ${projected.length} item-uri proiectate`);
    });
  }
}
```

### Comparatie: Decorators vs Signal Queries

| Aspect | `@ViewChild` / `@ContentChild` | `viewChild()` / `contentChild()` |
|--------|---------------------------------|----------------------------------|
| Tip returnat | Valoare directa (poate fi undefined) | `Signal<T \| undefined>` |
| Disponibilitate | Dupa `ngAfterViewInit` / `ngAfterContentInit` | Reactiv - se actualizeaza automat |
| Reactie la schimbari | `QueryList.changes` (Observable) | `effect()` nativ |
| Required | `{ static: true }` hack | `.required()` - type-safe |
| Integrare cu signals | Manuala | Nativa |
| Recomandat | Angular < 17 | Angular 17+ |

### Optiuni avansate pentru queries

```typescript
@Component({
  selector: 'app-advanced-queries',
  template: `
    <div #wrapper>
      <app-item />
    </div>
  `
})
export class AdvancedQueriesComponent {
  // read option - specifica CE sa citeasca din elementul gasit
  wrapperEl = viewChild('wrapper', { read: ElementRef });
  wrapperVcr = viewChild('wrapper', { read: ViewContainerRef });

  // descendants option (doar pentru contentChildren)
  // true = cauta in toti descendentii (default pentru contentChildren)
  // false = cauta doar in copiii directi
  deepItems = contentChildren(ItemComponent, { descendants: true });
  directItems = contentChildren(ItemComponent, { descendants: false });
}
```

---

## 5. Decoratori si Metadata

### 5.1 @Component

Defineste o componenta Angular - o clasa TypeScript asociata cu un template HTML si stiluri CSS.

```typescript
@Component({
  // Selector CSS folosit in template-uri
  selector: 'app-user-card',

  // In Angular v20+ do NOT set standalone: true (default). In v17–18 it was explicit.

  // Dependentele componentei (alte componente, directive, pipe-uri)
  imports: [DatePipe, RouterLink, MatButtonModule],

  // Template inline sau extern
  template: `<div>{{ user().name }}</div>`,
  // SAU: templateUrl: './user-card.component.html',

  // Stiluri inline sau externe (ViewEncapsulation)
  styles: [`
    :host { display: block; }
    .card { border: 1px solid #ccc; }
  `],
  // SAU: styleUrls: ['./user-card.component.scss'],
  // SAU in Angular 17+: styleUrl: './user-card.component.scss', (singular)

  // Encapsulare stiluri
  encapsulation: ViewEncapsulation.Emulated, // default
  // Emulated - simuleaza Shadow DOM cu atribute unice
  // ShadowDom - foloseste Shadow DOM nativ
  // None - stilurile sunt globale

  // Strategia de Change Detection
  changeDetection: ChangeDetectionStrategy.OnPush,
  // OnPush - componenta se verifica DOAR cand:
  //   1. Un @Input() primeste o referinta noua
  //   2. Un eveniment DOM (click, keyup etc.) apare in componenta
  //   3. Un signal folosit in template se schimba
  //   4. markForCheck() este apelat manual
  //   5. Un Observable cu pipe async emite

  // Animatii
  animations: [fadeInAnimation],

  // Host: use host object; do NOT use @HostBinding/@HostListener (angular.dev style guide)
  host: {
    'class': 'user-card',
    '[class.active]': 'isActive()',
    '(click)': 'onClick($event)',
    '[attr.role]': '"article"',
    '[attr.aria-label]': 'user().name'
  },

  // Providers locali (instanta per componenta)
  providers: [UserCardService]
})
export class UserCardComponent {
  user = input.required<User>();
  isActive = input(false);

  onClick(event: MouseEvent): void { /* ... */ }
}
```

### 5.2 @Directive

Similar cu `@Component`, dar fara template. Modifica comportamentul elementelor existente.

```typescript
// Directiva de atribut - modifica aspect/comportament
@Directive({
  selector: '[appTooltip]',
  host: {
    '(mouseenter)': 'show()',
    '(mouseleave)': 'hide()'
  }
})
export class TooltipDirective {
  appTooltip = input.required<string>();

  private el = inject(ElementRef);
  private renderer = inject(Renderer2);
  private tooltipEl: HTMLElement | null = null;

  show(): void {
    this.tooltipEl = this.renderer.createElement('span');
    this.renderer.addClass(this.tooltipEl, 'tooltip');
    const text = this.renderer.createText(this.appTooltip());
    this.renderer.appendChild(this.tooltipEl, text);
    this.renderer.appendChild(this.el.nativeElement, this.tooltipEl);
  }

  hide(): void {
    if (this.tooltipEl) {
      this.renderer.removeChild(this.el.nativeElement, this.tooltipEl);
      this.tooltipEl = null;
    }
  }
}

// Directiva de host - se ataseaza la componente
@Directive({
  selector: 'app-button[appLoadingState]'
})
export class LoadingStateDirective {
  isLoading = input.required<boolean>({ alias: 'appLoadingState' });

  private button = inject(ButtonComponent);

  constructor() {
    effect(() => {
      this.button.disabled = this.isLoading();
      this.button.showSpinner = this.isLoading();
    });
  }
}
```

### 5.3 @Pipe

Transforma date in template-uri. Poate fi `pure` (default, memoizat) sau `impure`.

```typescript
@Pipe({
  name: 'timeAgo',
  pure: true // default - se recalculeaza doar cand input-ul se schimba
})
export class TimeAgoPipe implements PipeTransform {
  transform(value: Date | string | number, locale: string = 'ro'): string {
    const date = new Date(value);
    const now = new Date();
    const diff = now.getTime() - date.getTime();
    const seconds = Math.floor(diff / 1000);
    const minutes = Math.floor(seconds / 60);
    const hours = Math.floor(minutes / 60);
    const days = Math.floor(hours / 24);

    if (seconds < 60) return 'chiar acum';
    if (minutes < 60) return `acum ${minutes} minute`;
    if (hours < 24) return `acum ${hours} ore`;
    if (days < 30) return `acum ${days} zile`;
    return date.toLocaleDateString(locale);
  }
}

// Utilizare: {{ createdAt | timeAgo }}
// Utilizare cu parametru: {{ createdAt | timeAgo:'en' }}
```

```typescript
// Pipe impure - se recalculeaza la FIECARE change detection cycle
// ATENTIE: impact pe performanta! Foloseste cu precautie.
@Pipe({
  name: 'filterActive',
  pure: false // se recalculeaza la fiecare CD cycle
})
export class FilterActivePipe implements PipeTransform {
  transform(items: any[]): any[] {
    return items?.filter(item => item.active) ?? [];
  }
}

// In Angular modern, pipe-urile impure sunt rar necesare.
// Alternativa: computed signal
// filteredItems = computed(() => this.items().filter(i => i.active));
```

### 5.4 @Injectable

Marcheaza o clasa ca disponibila pentru dependency injection.

```typescript
@Injectable({
  providedIn: 'root' // singleton la nivel de aplicatie (tree-shakeable)
})
export class AuthService {
  private http = inject(HttpClient);
  private router = inject(Router);

  currentUser = signal<User | null>(null);
  isAuthenticated = computed(() => this.currentUser() !== null);

  login(credentials: LoginRequest): Observable<User> {
    return this.http.post<User>('/api/auth/login', credentials).pipe(
      tap(user => this.currentUser.set(user))
    );
  }

  logout(): void {
    this.currentUser.set(null);
    this.router.navigate(['/login']);
  }
}

// Alte optiuni de providedIn:
@Injectable({ providedIn: 'root' })     // singleton global (recomandat)
@Injectable({ providedIn: 'platform' }) // partajat intre aplicatii (micro-frontends)
@Injectable({ providedIn: 'any' })      // instanta noua per lazy module
@Injectable()                            // trebuie adaugat manual in providers[]
```

### 5.5 @Input(), @Output() si model()

```typescript
@Component({
  selector: 'app-search',
  template: `
    <input
      [value]="query()"
      (input)="onInput($event)"
      (keyup.enter)="search.emit(query())"
    />
    <button (click)="clear()">X</button>
  `
})
export class SearchComponent {
  // TRADITIONAL: @Input si @Output
  // @Input() query: string = '';
  // @Output() queryChange = new EventEmitter<string>();
  // @Output() search = new EventEmitter<string>();

  // MODERN (Angular 17+): Signal inputs
  // input() - optional cu valoare default
  placeholder = input('Cauta...');

  // input.required() - obligatoriu
  // minLength = input.required<number>();

  // output() - inlocuieste @Output + EventEmitter
  search = output<string>();

  // model() - two-way binding cu signals (inlocuieste Input + Output pattern)
  // Creeaza automat un signal writable + output queryChange
  query = model('');            // optional cu default
  // query = model.required<string>(); // obligatoriu

  onInput(event: Event): void {
    const value = (event.target as HTMLInputElement).value;
    this.query.set(value); // actualizeaza si emite queryChange automat
  }

  clear(): void {
    this.query.set('');
  }
}

// Utilizare cu two-way binding:
// <app-search [(query)]="searchTerm" (search)="onSearch($event)" />
// searchTerm este un WritableSignal in parinte
```

### Comparatie: API-ul clasic vs modern

```typescript
// -------- CLASIC (Angular < 17) --------
@Component({ /* ... */ })
export class ClassicComponent {
  @Input() name: string = '';
  @Input({ required: true }) id!: number;
  @Input({ transform: booleanAttribute }) disabled: boolean = false;
  @Output() save = new EventEmitter<Data>();
  @Output() nameChange = new EventEmitter<string>(); // two-way: [(name)]
}

// -------- MODERN (Angular 17+) --------
@Component({ /* ... */ })
export class ModernComponent {
  name = input('');                                      // Signal<string>
  id = input.required<number>();                         // InputSignal<number>
  disabled = input(false, { transform: booleanAttribute }); // Signal<boolean>
  save = output<Data>();                                 // OutputEmitterRef<Data>
  nameModel = model('');                                 // ModelSignal<string> (two-way)
}
```

---

## 6. Angular Compiler

### 6.1 AOT vs JIT Compilation

Angular transforma template-urile HTML si decoratorii TypeScript in cod JavaScript executabil. Exista doua moduri de compilare:

| Aspect | AOT (Ahead-of-Time) | JIT (Just-in-Time) |
|--------|---------------------|---------------------|
| Cand | La build time | La runtime (in browser) |
| Default | Da (din Angular 9+) | Nu (doar dev cu `ng serve --configuration=development`) |
| Bundle size | Mai mic (compilatorul NU e inclus) | Mai mare (compilatorul e in bundle) |
| Erori template | La build time | La runtime |
| Startup | Rapid | Lent (compilare in browser) |
| Securitate | Template-urile sunt pre-compilate | Template-urile pot fi exploatate |
| Comanda | `ng build` | `ng serve` (dev only) |

```bash
# AOT (productie) - default
ng build --configuration=production

# JIT este deprecat in Angular modern si nu ar trebui folosit in productie
# Dar e inca disponibil pentru development:
ng serve  # foloseste esbuild + AOT chiar si in dev (Angular 17+)
```

### 6.2 Ivy Renderer

Ivy este motorul de randare Angular incepand cu **Angular 9** (default). Inlocuieste View Engine (deprecated, eliminat in Angular 16).

**Principii cheie Ivy:**

1. **Locality principle** - fiecare componenta se compileaza independent (nu depinde de alte componente/module la compilare)
2. **Tree-shakeable** - codul Angular nefolosit este eliminat din bundle
3. **Incremental compilation** - doar componentele modificate sunt recompilate

```typescript
// Ce face Ivy "sub capota":
// Template-ul:
// <div>{{ name() }}</div>
// <app-child [data]="items()" />

// Este compilat in instructiuni Ivy (simplificat):
function UserComponent_Template(rf, ctx) {
  if (rf & RenderFlags.Create) {
    // Instructiuni de creare
    elementStart(0, 'div');
    text(1);
    elementEnd();
    element(2, 'app-child');
  }
  if (rf & RenderFlags.Update) {
    // Instructiuni de update
    advance(1);
    textInterpolate(ctx.name());
    advance(1);
    property('data', ctx.items());
  }
}
```

**Beneficii Ivy:**
- Build-uri mai rapide (compilare incrementala)
- Bundle-uri mai mici (tree-shaking agresiv)
- Debugging mai bun (`ng.getComponent(element)` in consola browser)
- Standalone components (posibile datorita locality principle)
- Lazy loading la nivel de componenta

### 6.3 Template Type Checking

Angular verifica tipurile din template-uri la compilare (AOT). Exista trei niveluri de strictete:

```json
// tsconfig.json
{
  "angularCompilerOptions": {
    // Nivel 1: Basic (default inainte de Angular 12)
    "fullTemplateTypeCheck": true,

    // Nivel 2: Strict (default Angular 12+)
    "strictTemplates": true,
    // Echivalent cu TOATE acestea pe true:
    // "strictInputTypes": true,
    // "strictInputAccessModifiers": true,
    // "strictNullInputTypes": true,
    // "strictAttributeTypes": true,
    // "strictSafeNavigationTypes": true,
    // "strictDomLocalRefTypes": true,
    // "strictOutputEventTypes": true,
    // "strictDomEventTypes": true,
    // "strictContextGenerics": true
  }
}
```

**Exemple de erori detectate de `strictTemplates`:**

```typescript
@Component({
  template: `
    <!-- EROARE: Type 'string' is not assignable to type 'number' -->
    <app-counter [count]="userName" />

    <!-- EROARE: Property 'fullName' does not exist on type 'User' -->
    <span>{{ user().fullName }}</span>

    <!-- EROARE: Object is possibly 'undefined' -->
    <span>{{ user().address.street }}</span>

    <!-- CORECT: safe navigation -->
    <span>{{ user()?.address?.street }}</span>

    <!-- $any() - escape hatch (evita daca posibil) -->
    <span>{{ $any(user()).legacyField }}</span>
  `
})
export class StrictComponent {
  userName = signal('John');
  user = signal<User | undefined>(undefined);
}
```

---

## 7. Angular CLI si Schematics

### 7.1 Comenzi esentiale ng generate

```bash
# Componenta standalone (default in Angular 19)
ng generate component features/user-profile
# Creeaza: user-profile.component.ts, .html, .scss, .spec.ts

# Componenta inline (fara fisiere separate)
ng g c shared/button --inline-template --inline-style

# Serviciu (providedIn: 'root' default)
ng g service core/services/auth

# Directiva
ng g directive shared/directives/tooltip

# Pipe
ng g pipe shared/pipes/time-ago

# Guard (functional guard - default in Angular modern)
ng g guard core/guards/auth
# Intreaba tipul: CanActivate, CanDeactivate, etc.

# Interceptor (functional)
ng g interceptor core/interceptors/auth

# Modul (rar folosit in Angular 19, dar exista)
ng g module features/admin --routing

# Clasa, interfata, enum
ng g class models/user
ng g interface models/user
ng g enum models/status

# Library (pentru monorepo / design systems)
ng g library @myorg/ui-components

# Schematic specific Angular Material
ng g @angular/material:table features/data-table
ng g @angular/material:navigation shared/layout
```

### 7.2 ng update

```bash
# Verifica ce actualizari sunt disponibile
ng update

# Actualizeaza Angular core + CLI
ng update @angular/core @angular/cli

# Actualizeaza Angular Material
ng update @angular/material

# Migrare majora (ex: Angular 18 -> 19)
ng update @angular/core@19 @angular/cli@19

# Forteaza actualizarea (cu grija!)
ng update @angular/core --force

# ng update ruleaza automat schematics de migrare:
# - Actualizeaza package.json
# - Modifica codul sursa pentru breaking changes
# - Actualizeaza configuratia (angular.json, tsconfig)
```

### 7.3 Custom Schematics - Introducere

Schematics sunt generatoare de cod care transforma proiecte Angular. Sunt folosite intern de Angular CLI si pot fi create custom.

```bash
# Creeaza un proiect de schematics
npm install -g @angular-devkit/schematics-cli
schematics blank --name=my-schematics
```

**Structura unui schematic:**

```
my-schematics/
  src/
    my-generator/
      index.ts          # Factory function
      schema.json        # Schema optiunilor
      schema.d.ts        # Tipuri TypeScript
      files/             # Template-uri de fisiere
        __name@dasherize__/
          __name@dasherize__.component.ts.template
```

```typescript
// src/my-generator/index.ts
import { Rule, SchematicContext, Tree, apply, url, template,
         mergeWith, move } from '@angular-devkit/schematics';
import { strings } from '@angular-devkit/core';

export interface MyGeneratorSchema {
  name: string;
  path?: string;
}

export function myGenerator(options: MyGeneratorSchema): Rule {
  return (tree: Tree, context: SchematicContext) => {
    // Genereaza fisiere din template-uri
    const templateSource = apply(url('./files'), [
      template({
        ...strings,           // dasherize, classify, camelize etc.
        ...options,
      }),
      move(options.path || 'src/app'),
    ]);

    return mergeWith(templateSource)(tree, context);
  };
}
```

```json
// src/my-generator/schema.json
{
  "$schema": "http://json-schema.org/schema",
  "id": "MyGeneratorSchema",
  "title": "My Generator Schema",
  "type": "object",
  "properties": {
    "name": {
      "type": "string",
      "description": "Numele componentei",
      "$default": {
        "$source": "argv",
        "index": 0
      },
      "x-prompt": "Cum doresti sa se numeasca componenta?"
    },
    "path": {
      "type": "string",
      "description": "Calea unde sa fie generat"
    }
  },
  "required": ["name"]
}
```

```bash
# Rulare schematic custom
ng generate my-schematics:my-generator my-feature

# Dry run (preview fara modificari)
ng generate my-schematics:my-generator my-feature --dry-run
```

**Cazuri de utilizare practice pentru schematics custom:**
- Generare pagini cu layout predefinit (header, sidebar, content)
- Generare feature modules cu structura standard (component + service + routes + tests)
- Adaugare automata de boilerplate NGRX (actions, reducers, effects, selectors)
- Enforcing architectural patterns (ex: structura de foldere)

---

## 8. Noul Control Flow (Angular 17+)

Angular 17 introduce o sintaxa noua de control flow, direct in template, care inlocuieste directivele structurale `*ngIf`, `*ngFor`, `*ngSwitch`. Este mai performanta si nu necesita import de module.

### 8.1 @if / @else if / @else

```html
<!-- VECHI: *ngIf -->
<div *ngIf="user; else loadingTemplate">
  <span>{{ user.name }}</span>
</div>
<ng-template #loadingTemplate>
  <app-spinner />
</ng-template>

<!-- NOU: @if -->
@if (user(); as u) {
  <span>{{ u.name }}</span>
} @else if (isLoading()) {
  <app-spinner />
} @else {
  <p>Niciun utilizator gasit</p>
}
```

**Avantaje fata de `*ngIf`:**
- `@else if` nativ (imposibil cu `*ngIf`)
- Nu necesita `<ng-template>` pentru else
- `as` variabila locala functioneaza natural
- Nu necesita import `CommonModule` / `NgIf`
- Performanta mai buna (compilat direct, fara directive overhead)

### 8.2 @for (cu track obligatoriu)

```html
<!-- VECHI: *ngFor -->
<ul>
  <li *ngFor="let item of items; let i = index; let first = first; trackBy: trackById">
    {{ i }}: {{ item.name }}
  </li>
</ul>

<!-- NOU: @for -->
<ul>
  @for (item of items(); track item.id; let i = $index, first = $first) {
    <li [class.first]="first">
      {{ i }}: {{ item.name }}
    </li>
  } @empty {
    <li>Lista este goala</li>
  }
</ul>
```

**Variabile contextuale disponibile in `@for`:**

| Variabila | Tip | Descriere |
|-----------|-----|-----------|
| `$index` | `number` | Indexul curent (de la 0) |
| `$first` | `boolean` | Primul element |
| `$last` | `boolean` | Ultimul element |
| `$even` | `boolean` | Index par |
| `$odd` | `boolean` | Index impar |
| `$count` | `number` | Numarul total de elemente |

**`track` este OBLIGATORIU** (spre deosebire de `trackBy` care era optional in `*ngFor`):

```html
<!-- Track dupa proprietate -->
@for (user of users(); track user.id) { ... }

<!-- Track dupa index (pentru liste fara ID unic - mai putin performant) -->
@for (item of items(); track $index) { ... }

<!-- Track dupa elementul insusi (pentru primitive) -->
@for (name of names(); track name) { ... }

<!-- Track compus -->
@for (item of items(); track item.category + '-' + item.id) { ... }
```

**`@empty` block** - se afiseaza cand colectia este goala (imposibil cu `*ngFor`):

```html
@for (notification of notifications(); track notification.id) {
  <app-notification-card [notification]="notification" />
} @empty {
  <div class="empty-state">
    <img src="assets/no-notifications.svg" />
    <p>Nu ai notificari noi</p>
  </div>
}
```

### 8.3 @switch

```html
<!-- VECHI: ngSwitch -->
<div [ngSwitch]="status">
  <span *ngSwitchCase="'active'">Activ</span>
  <span *ngSwitchCase="'inactive'">Inactiv</span>
  <span *ngSwitchCase="'pending'">In asteptare</span>
  <span *ngSwitchDefault>Necunoscut</span>
</div>

<!-- NOU: @switch -->
@switch (user().role) {
  @case ('admin') {
    <app-admin-dashboard />
  }
  @case ('editor') {
    <app-editor-panel />
  }
  @case ('viewer') {
    <app-viewer-panel />
  }
  @default {
    <app-access-denied />
  }
}
```

**Avantaje fata de `ngSwitch`:**
- Sintaxa mai clara si concisa
- Nu necesita element container (`[ngSwitch]`)
- Nu necesita import `CommonModule` / `NgSwitch`
- Type checking mai bun pe case-uri

**Style guide (angular.dev, v20+):** Preferă `[class]` / `[style]` bindings în loc de `ngClass` / `ngStyle` (performanță și consistență cu HTML).

### 8.4 @defer (Deferrable Views)

Desi nu e strict "control flow", `@defer` a fost introdus in acelasi timp si este esential:

```html
<!-- Lazy loading la nivel de template -->
@defer (on viewport) {
  <!-- Componenta se incarca DOAR cand intra in viewport -->
  <app-heavy-chart [data]="chartData()" />
} @placeholder {
  <!-- Afisat inainte de trigger -->
  <div class="chart-placeholder">Grafic disponibil</div>
} @loading (minimum 500ms) {
  <!-- Afisat in timpul incarcarii -->
  <app-spinner />
} @error {
  <!-- Afisat daca incarcarea esueaza -->
  <p>Eroare la incarcarea graficului</p>
}

<!-- Triggere disponibile: -->
@defer (on viewport)           { ... }  <!-- cand elementul e vizibil -->
@defer (on interaction)        { ... }  <!-- la click/focus/hover -->
@defer (on hover)              { ... }  <!-- la hover -->
@defer (on idle)               { ... }  <!-- cand browser-ul e idle -->
@defer (on timer(3s))          { ... }  <!-- dupa 3 secunde -->
@defer (on immediate)          { ... }  <!-- imediat, dar lazy loaded -->
@defer (when isVisible())      { ... }  <!-- conditie custom -->
@defer (on viewport; prefetch on idle) { ... }  <!-- prefetch separat -->
```

### 8.5 Migrarea de la directive la noul control flow

```bash
# Angular CLI ofera un schematic de migrare automata:
ng generate @angular/core:control-flow

# Converteste automat:
# *ngIf       -> @if
# *ngFor      -> @for (adauga track $index daca nu exista trackBy)
# *ngSwitch   -> @switch
```

---

## Noutati Angular 20-21 (cu exemple inainte/dupa)

### Angular 20 (Mai 2025)

#### Template literals in templates

```typescript
// INAINTE (Angular 19): concatenare cu +
@Component({
  template: `<h1>{{ 'Hello, ' + name() + '!' }}</h1>`
})

// DUPA (Angular 20): template literals native
@Component({
  template: `<h1>{{ \`Hello, \${name()}!\` }}</h1>`
})
// Functioneaza si cu expresii complexe:
// {{ `${firstName()} ${lastName()} (${role()})` }}
```

#### Exponentiation operator

```typescript
// INAINTE (Angular 19): Math.pow() sau pipe custom
@Component({
  template: `
    <p>Suprafata cercului: {{ 3.14 * radius() * radius() }}</p>
    <!-- sau trebuia un pipe custom / metoda in component -->
    <p>Suprafata: {{ calculateArea(radius()) }}</p>
  `
})
export class OldComponent {
  calculateArea(r: number): number { return Math.PI * Math.pow(r, 2); }
}

// DUPA (Angular 20): ** direct in template
@Component({
  template: `
    <p>Suprafata cercului: {{ 3.14 * radius() ** 2 }}</p>
    <p>Cub: {{ value() ** 3 }}</p>
  `
})
```

#### `in` keyword in templates

```typescript
// INAINTE (Angular 19): verificare cu metoda in component sau hasOwnProperty
@Component({
  template: `
    @if (hasDiscount(product())) {
      <span class="discount">-{{ product().discount }}%</span>
    }
  `
})
export class OldComponent {
  hasDiscount(product: any): boolean {
    return 'discount' in product;
  }
}

// DUPA (Angular 20): `in` direct in template
@Component({
  template: `
    @if ('discount' in product()) {
      <span class="discount">-{{ product().discount }}%</span>
    }
  `
})
```

#### `void` operator in templates

```typescript
// INAINTE (Angular 19): event handler-ul returna o valoare
// -> daca returna false, browser-ul facea preventDefault() implicit
@Component({
  template: `
    <!-- PROBLEMA: daca doSomething() returneaza false, click-ul e prevenit -->
    <a href="/page" (click)="doSomething()">Link</a>

    <!-- WORKAROUND: adaugai ; false sau $event.stopPropagation() separat -->
    <a href="/page" (click)="doSomething(); $event.stopPropagation()">Link</a>
  `
})

// DUPA (Angular 20): void previne valoarea returnata
@Component({
  template: `
    <!-- void ignora valoarea returnata -> nu mai face preventDefault() -->
    <a href="/page" (click)="void doSomething()">Link</a>
  `
})
```

#### Host binding type checking

```typescript
// INAINTE (Angular 19): host bindings nu erau validate de TypeScript
@Component({
  host: {
    '[class.actve]': 'isActive()',       // typo in 'active' -> FARA EROARE!
    '[attr.role]': 'getRole()',          // getRole() nu exista -> FARA EROARE!
    '(click)': 'onClck($event)',        // typo in 'onClick' -> FARA EROARE!
  }
})

// DUPA (Angular 20): validare completa TypeScript
@Component({
  host: {
    '[class.actve]': 'isActive()',       // tot fara eroare (CSS class, nu TS)
    '[attr.role]': 'getRole()',          // EROARE: Property 'getRole' does not exist
    '(click)': 'onClck($event)',        // EROARE: Property 'onClck' does not exist
  }
})
```

#### `ng-reflect` attributes eliminate

```html
<!-- INAINTE (Angular 19 dev mode): Angular adauga atribute ng-reflect-* -->
<app-user ng-reflect-name="John" ng-reflect-age="25">
  <!-- Vizibil in DOM inspector, polua DOM-ul, impact pe performanta -->
</app-user>

<!-- DUPA (Angular 20): ng-reflect-* NU mai sunt emise nici in dev mode -->
<app-user>
  <!-- DOM-ul e curat; foloseste Angular DevTools pentru debugging -->
</app-user>
```

#### afterEveryRender / afterNextRender stabilizate

```typescript
// INAINTE (Angular 18-19): afterRender era in developer preview
import { afterRender, afterNextRender } from '@angular/core';

@Component({})
export class OldComponent {
  constructor() {
    afterNextRender(() => {
      // afterRender era developer preview, putea avea breaking changes
      this.initChart();
    });
    afterRender(() => {
      // Se apela dupa FIECARE renderizare
      this.syncScroll();
    });
  }
}

// DUPA (Angular 20): API stabil, redenumit afterEveryRender
import { afterEveryRender, afterNextRender } from '@angular/core';

@Component({})
export class NewComponent {
  constructor() {
    afterNextRender(() => {
      this.initChart(); // O singura data, dupa prima renderizare
    });
    afterEveryRender(() => {
      this.syncScroll(); // Dupa FIECARE renderizare (fost afterRender)
    });
  }
}
```

#### HammerJS deprecat oficial

```typescript
// INAINTE (Angular 19): HammerJS pentru gesture events
// package.json: "hammerjs": "^2.0.8"
import 'hammerjs';

@NgModule({
  imports: [HammerModule]  // importat in app module
})

@Component({
  template: `
    <div (swipeleft)="onSwipeLeft()"
         (swiperight)="onSwipeRight()"
         (pinch)="onPinch($event)">
      Swipe me
    </div>
  `
})

// DUPA (Angular 20): HammerJS deprecat, foloseste Pointer Events
@Component({
  template: `
    <div (pointerdown)="onPointerDown($event)"
         (pointermove)="onPointerMove($event)"
         (pointerup)="onPointerUp($event)">
      Swipe me
    </div>
  `
})
export class NewComponent {
  // Implementare custom swipe cu Pointer Events API
  private startX = 0;

  onPointerDown(e: PointerEvent) { this.startX = e.clientX; }
  onPointerUp(e: PointerEvent) {
    const diff = e.clientX - this.startX;
    if (diff > 50) this.onSwipeRight();
    if (diff < -50) this.onSwipeLeft();
  }
}
```

---

### Angular 21 (Noiembrie 2025)

#### Signal Forms (experimental)

```typescript
// INAINTE (Angular 19): Reactive Forms
import { ReactiveFormsModule, FormBuilder, Validators } from '@angular/forms';

@Component({
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="loginForm" (ngSubmit)="onSubmit()">
      <input formControlName="email" />
      <input formControlName="password" type="password" />
      @if (loginForm.get('email')?.errors?.['required'] &&
           loginForm.get('email')?.touched) {
        <span>Email obligatoriu</span>
      }
      @if (loginForm.get('email')?.errors?.['email']) {
        <span>Email invalid</span>
      }
      <button type="submit" [disabled]="loginForm.invalid || submitting">
        {{ submitting ? 'Se trimite...' : 'Login' }}
      </button>
    </form>
  `
})
export class LoginComponent {
  private fb = inject(FormBuilder);
  submitting = false;

  loginForm = this.fb.group({
    email: ['', [Validators.required, Validators.email]],
    password: ['', [Validators.required, Validators.minLength(8)]]
  });

  async onSubmit() {
    if (this.loginForm.invalid) return;
    this.submitting = true;
    try {
      await this.authService.login(this.loginForm.value);
    } finally {
      this.submitting = false;
    }
  }
}

// DUPA (Angular 21): Signal Forms
import { form, FormField, required, email, minLength, submit } from '@angular/forms/signals';

@Component({
  imports: [FormField],
  template: `
    <input [formField]="loginForm.email">
    <input [formField]="loginForm.password" type="password">
    @if (loginForm.email().errors().length) {
      <span>Email invalid</span>
    }
    <button (click)="onSubmit()" [disabled]="loginForm().submitting()">
      {{ loginForm().submitting() ? 'Se trimite...' : 'Login' }}
    </button>
  `
})
export class LoginComponent {
  private model = signal({ email: '', password: '' });
  loginForm = form(this.model, (f) => {
    required(f.email);
    email(f.email);
    required(f.password);
    minLength(f.password, 8);
  });

  async onSubmit() {
    await submit(this.loginForm, async (form) => {
      await this.authService.login(form().value());
    });
    // submit() gestioneaza automat: validare, submitting state, prevent double-submit
  }
}
```

#### Zoneless default

```typescript
// INAINTE (Angular 18-19): Zone.js inclus automat
// polyfills.ts (Angular < 17) sau angular.json
import 'zone.js';

// main.ts - Zone.js era mecanismul implicit de change detection
bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes),
    provideHttpClient(),
    // Zone.js patch-uia setTimeout, Promise, addEventListener etc.
    // Change detection rula la ORICE event async, chiar irelevant
  ]
});

// OPTIONAL in Angular 19: puteai opta pentru zoneless explicit
bootstrapApplication(AppComponent, {
  providers: [
    provideZonelessChangeDetection(), // opt-in manual
  ]
});

// DUPA (Angular 21): zoneless este DEFAULT pentru aplicatii noi (ng new)
// Zone.js NU mai e inclus in proiecte noi
bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes),
    // provideZonelessChangeDetection() -- nu mai trebuie, e default!
    // Change detection se bazeaza pe signals + events din template
  ]
});
// Consecinta: setTimeout(() => this.name = 'test', 1000) NU mai
// triggeruieste change detection! Trebuie signals:
// setTimeout(() => this.name.set('test'), 1000) -> OK
```

#### Vitest ca default test runner

```typescript
// INAINTE (Angular 17-19): Karma (default) sau Jest (community)
// karma.conf.js - configuratie complexa
module.exports = function(config) {
  config.set({
    frameworks: ['jasmine', '@angular-devkit/build-angular'],
    browsers: ['Chrome'],
    // ... zeci de linii de configurare
  });
};

// test.spec.ts cu Jasmine
describe('UserService', () => {
  let service: UserService;

  beforeEach(() => {
    TestBed.configureTestingModule({});
    service = TestBed.inject(UserService);
  });

  it('should return user', () => {
    expect(service.getUser()).toBeTruthy();
  });
});

// DUPA (Angular 21): Vitest stabil, default la ng new
// vitest.config.ts - configuratie minimala (generata automat)
// Sintaxa testelor e IDENTICA (Vitest e compatibil Jasmine/Jest)
describe('UserService', () => {
  let service: UserService;

  beforeEach(() => {
    TestBed.configureTestingModule({});
    service = TestBed.inject(UserService);
  });

  it('should return user', () => {
    expect(service.getUser()).toBeTruthy();
  });
});
// Diferenta: Vitest e ~10x mai rapid, suporta ESM nativ,
// watch mode instant, si nu necesita browser real pentru unit tests
```

#### Angular Aria (componente headless accesibile)

```typescript
// INAINTE (Angular 19): Angular Material SAU implementare manuala ARIA
// Varianta 1: Angular Material - vine cu stiluri Material Design
import { MatTabsModule } from '@angular/material/tabs';
@Component({
  imports: [MatTabsModule],
  template: `
    <mat-tab-group>
      <mat-tab label="Tab 1">Continut 1</mat-tab>
      <mat-tab label="Tab 2">Continut 2</mat-tab>
    </mat-tab-group>
  `
  // PROBLEMA: stilurile Material Design sunt greu de suprascris
})

// Varianta 2: Manual ARIA - mult boilerplate
@Component({
  template: `
    <div role="tablist">
      @for (tab of tabs(); track tab.id; let i = $index) {
        <button role="tab"
                [attr.aria-selected]="selectedIndex() === i"
                [attr.aria-controls]="'panel-' + tab.id"
                [id]="'tab-' + tab.id"
                [tabindex]="selectedIndex() === i ? 0 : -1"
                (click)="select(i)"
                (keydown)="onKeydown($event, i)">
          {{ tab.label }}
        </button>
      }
    </div>
    @for (tab of tabs(); track tab.id; let i = $index) {
      <div role="tabpanel"
           [id]="'panel-' + tab.id"
           [attr.aria-labelledby]="'tab-' + tab.id"
           [hidden]="selectedIndex() !== i">
        <!-- continut -->
      </div>
    }
  `
})
export class ManualTabsComponent {
  // Trebuia sa implementezi manual: keyboard navigation (Arrow keys),
  // focus management, ARIA attributes, Home/End support...
  onKeydown(event: KeyboardEvent, index: number) {
    // ~50 linii de keyboard handling manual
  }
}

// DUPA (Angular 21): Angular Aria - headless, accesibil by default
import { CdkTabs, CdkTabList, CdkTab, CdkTabPanel } from '@angular/cdk-aria';
@Component({
  imports: [CdkTabs, CdkTabList, CdkTab, CdkTabPanel],
  template: `
    <div cdkTabs>
      <div cdkTabList>
        @for (tab of tabs(); track tab.id) {
          <button cdkTab>{{ tab.label }}</button>
        }
      </div>
      @for (tab of tabs(); track tab.id) {
        <div cdkTabPanel>{{ tab.content }}</div>
      }
    </div>
  `
  // ARIA roles, keyboard navigation, focus management -> AUTOMAT
  // ZERO stiluri impuse -> aplici propriul design system
})
```

#### Generic SimpleChanges

```typescript
// INAINTE (Angular 19): SimpleChanges fara type safety
@Component({})
export class OldComponent implements OnChanges {
  @Input() name = '';
  @Input() age = 0;

  ngOnChanges(changes: SimpleChanges): void {
    // changes['name'] -> SimpleChange (untyped)
    // changes['nmae'] -> compileaza fara eroare (typo!)
    if (changes['name']) {
      const prev: any = changes['name'].previousValue; // any!
      const curr: any = changes['name'].currentValue;  // any!
      console.log(prev, curr);
    }
    // Puteai accesa proprietati inexistente fara eroare:
    if (changes['nonExistent']) { /* compileaza OK, dar nu face nimic */ }
  }
}

// DUPA (Angular 21): SimpleChanges<T> cu type safety
@Component({})
export class NewComponent implements OnChanges {
  @Input() name = '';
  @Input() age = 0;

  ngOnChanges(changes: SimpleChanges<NewComponent>): void {
    // changes['name'] -> SimpleChange<string> (typed!)
    // changes['nmae'] -> EROARE TypeScript: typo detectat!
    if (changes['name']) {
      const prev: string = changes['name'].previousValue; // string!
      const curr: string = changes['name'].currentValue;  // string!
      console.log(prev, curr);
    }
    // changes['nonExistent'] -> EROARE: nu exista pe NewComponent
  }
}
```

#### HttpClient provided by default

```typescript
// INAINTE (Angular 19): trebuia sa adaugi provideHttpClient() manual
// main.ts
bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes),
    provideHttpClient(withInterceptors([authInterceptor])),  // OBLIGATORIU
    // Daca uitai -> "NullInjectorError: No provider for HttpClient!"
  ]
});

// DUPA (Angular 21): HttpClient e furnizat automat
// main.ts
bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes),
    // HttpClient e disponibil automat! Nu mai trebuie provideHttpClient()
    // Dar daca ai interceptori, tot trebuie sa-i configurezi:
    provideHttpClient(withInterceptors([authInterceptor])),
  ]
});
// Practic, inject(HttpClient) functioneaza "out of the box" fara configurare
```

#### ngClass / ngStyle -> [class] / [style] migration

```html
<!-- INAINTE (Angular 19): ngClass si ngStyle directive -->
<div [ngClass]="{'active': isActive(), 'disabled': isDisabled()}">
  Styled element
</div>
<div [ngClass]="dynamicClasses()">...</div>
<div [ngStyle]="{'color': textColor(), 'font-size.px': fontSize()}">
  Styled text
</div>

<!-- DUPA (Angular 21): [class] si [style] bindings native -->
<div [class.active]="isActive()" [class.disabled]="isDisabled()">
  Styled element
</div>
<div [class]="dynamicClasses()">...</div>
<div [style.color]="textColor()" [style.font-size.px]="fontSize()">
  Styled text
</div>

<!-- Schematics de migrare automata: -->
<!-- ng generate @angular/core:ngclass-to-class -->
<!-- ng generate @angular/core:ngstyle-to-style -->

<!-- Avantaje [class]/[style] vs ngClass/ngStyle:
     - Nu necesita import CommonModule
     - Performanta mai buna (binding direct, fara directiva)
     - Consistent cu HTML standard
     - Recomandat de Angular Style Guide v20+ -->
```

#### HttpResponse.responseType

```typescript
// INAINTE (Angular 19): nu puteai afla tipul raspunsului fetch
const response = await fetch('/api/data');
// response.type -> 'basic', 'cors', 'opaque' (doar pe Response nativ)
// HttpClient nu expunea aceasta informatie

this.http.get('/api/data', { observe: 'response' }).subscribe(res => {
  // res.headers, res.status -> DA
  // res.type -> NU EXISTA
});

// DUPA (Angular 21): HttpResponse.responseType disponibil
this.http.get('/api/data', { observe: 'response' }).subscribe(res => {
  console.log(res.responseType); // 'basic' | 'cors' | 'opaque' | ...
  // Util pentru: debugging CORS, detectare opaque responses, logging
});
```

---

## Intrebari frecvente de interviu

### 1. Care este diferenta dintre OnPush si Default change detection? Cum interactioneaza cu signals?

**Raspuns:** In strategia **Default**, Angular verifica componenta la fiecare ciclu de change detection (orice eveniment, timer, HTTP response). In **OnPush**, componenta se verifica doar cand: (a) un `@Input()` primeste o referinta noua (nu mutatie!), (b) un eveniment DOM apare in componenta, (c) `markForCheck()` este apelat, (d) un `async` pipe emite. **Cu signals**, Angular poate face change detection si mai granular - un signal care se schimba marcheaza automat componenta ca "dirty", chiar si cu OnPush, fara `markForCheck()`. In Angular 18+, signals permit eventual "signal-based change detection" care va face checkless rendering - doar componentele cu signals modificate se re-randeaza.

### 2. De ce este track obligatoriu in @for si cum afecteaza performanta?

**Raspuns:** `track` permite Angular sa identifice unic fiecare element din lista. Cand lista se schimba, Angular compara elementele vechi cu cele noi folosind cheia de track. Fara track corect, Angular ar trebui sa distruga si recreeze toate elementele DOM la fiecare schimbare, ceea ce este foarte costisitor. A fost facut obligatoriu in `@for` (fata de optional `trackBy` in `*ngFor`) pentru a preveni probleme de performanta frecvente. Cel mai bun track este o proprietate unica si stabila (ex: `item.id`). Track dupa `$index` este mai putin performant deoarece inserarea unui element la inceputul listei invalideaza toate indexurile.

### 3. Explica diferenta intre ViewChild si ContentChild. Cand devii disponibil fiecare?

**Raspuns:** **ViewChild** refera elemente din template-ul componentei (view). **ContentChild** refera elemente proiectate prin `<ng-content>` din exterior (content). ViewChild-urile devin disponibile in `ngAfterViewInit`, iar ContentChild-urile in `ngAfterContentInit` (care ruleaza **inainte**). Cu signal-based queries (`viewChild()`, `contentChild()`), disponibilitatea este reactiva - signal-ul returneaza `undefined` initial si se actualizeaza automat cand elementul devine disponibil, eliminand necesitatea de a astepta un lifecycle hook specific.

### 4. Ce este Ivy si care este principiul locality?

**Raspuns:** Ivy este motorul de compilare si randare Angular (default din Angular 9). **Locality principle** inseamna ca o componenta se compileaza folosind DOAR informatia proprie - decoratorul, template-ul, si metadatele componentelor importate direct. Nu mai depinde de contextul modulului. Aceasta a facut posibile: standalone components, compilare incrementala mai rapida, tree-shaking mai agresiv, si lazy loading la nivel de componenta. Inainte de Ivy (View Engine), compilatorul avea nevoie de contextul intregului NgModule.

### 5. Cum functioneaza content projection cu mai multe sloturi? Ce se intampla cu continutul care nu se potriveste niciunui selector?

**Raspuns:** Multi-slot projection foloseste atributul `select` pe `<ng-content>` cu selectori CSS (element, clasa, atribut). Angular distribuie continutul proiectat in slot-ul corespunzator. Continutul care **nu se potriveste niciunui selector** merge in slot-ul **default** (un `<ng-content>` fara `select`). Daca nu exista un slot default, continutul care nu se potriveste **nu se randeaza deloc**. `<ng-content>` nu creeaza elemente DOM proprii - este doar un punct de insertie.

### 6. Care este diferenta intre AOT si JIT? De ce este AOT preferat in productie?

**Raspuns:** **AOT** compileaza template-urile la build time, **JIT** la runtime in browser. AOT este preferat deoarece: (1) bundle-ul este mai mic (compilatorul Angular ~1MB nu este inclus), (2) erorile de template sunt detectate la build, (3) startup-ul este mai rapid (template-urile sunt deja compilate), (4) securitate mai buna (template-urile nu pot fi injectate la runtime). Din Angular 17+, chiar si `ng serve` foloseste AOT cu esbuild, facand JIT aproape complet irelevant.

### 7. Cum functioneaza @defer si care sunt implicatiile pentru performanta?

**Raspuns:** `@defer` permite lazy loading la nivel de template. Componenta din blocul `@defer` si toate dependentele sale sunt mutate automat intr-un chunk separat. Incarcarea se declanseaza la un trigger specificat (viewport, interaction, hover, timer, idle, conditie). Aceasta reduce dramatic dimensiunea bundle-ului initial si imbunatateste LCP (Largest Contentful Paint). `@placeholder` arata un continut static inainte de trigger, `@loading` in timpul incarcarii (cu optiuni `minimum` si `after` pentru a evita flicker), iar `@error` pentru erori. `prefetch` permite pre-incarcarea codului inainte de trigger.

### 8. Explica model() signals si cum difera de pattern-ul clasic Input + Output.

**Raspuns:** `model()` creeaza un **ModelSignal** care combina un input si un output intr-un singur API. Cand apelezi `model.set(value)`, valoarea se schimba local SI se emite automat un output `nameChange` (unde "name" este numele proprietatii). Aceasta permite two-way binding cu `[(name)]` fara boilerplate. Diferenta fata de clasic: (1) este un signal, deci reactiv si compatibil cu `computed()` si `effect()`, (2) nu necesita EventEmitter separat, (3) type-safe fara cast-uri. `model.required()` forteaza parintele sa furnizeze binding-ul.

### 9. Care este rolul ng-container si de ce este necesar?

**Raspuns:** `ng-container` este un **element logic** Angular care nu produce niciun nod DOM. Este necesar in trei situatii principale: (1) cand vrei sa aplici o directiva structurala fara a adauga un element DOM extra (ex: `*ngIf` pe o lista de `<li>` fara un `<div>` wrapper), (2) cand vrei sa aplici **doua directive structurale** (nu poti pune `*ngIf` si `*ngFor` pe acelasi element), (3) ca host pentru `ngTemplateOutlet`. Cu noul control flow (`@if`, `@for`), primele doua cazuri sunt rezolvate nativ, dar `ng-container` ramane util pentru template outlet si pentru a grupa elemente fara markup.

### 10. Cum ai migra o aplicatie Angular mare (50+ module) de la NgModule la standalone?

**Raspuns:** Migrarea trebuie facuta **incremental**, nu big-bang. Strategia: (1) Incepe cu `ng generate @angular/core:standalone` pe un feature module mic, verifica rezultatul. (2) Converteste mai intai componentele **leaf** (fara copii) - sunt cele mai simple. (3) Foloseste `importProvidersFrom()` in perioada de tranzitie pentru a partaja provideri intre lumea veche (NgModule) si cea noua (standalone). (4) Migreaza rutele la `loadComponent` pe masura ce convertesti componente. (5) Converteste core module (guards, interceptors) la functii standalone. (6) La final, converteste `AppModule` la `bootstrapApplication`. (7) In paralel, migreaza si control flow-ul cu `ng generate @angular/core:control-flow`. Fiecare pas trebuie validat cu teste existente.

### 11. Ce sunt Signal Forms din Angular 21 si cum difera de Reactive Forms?

**Raspuns:** Signal Forms sunt noul API experimental de formulare bazat pe signals, introdus in Angular 21. Diferentele cheie fata de Reactive Forms: (1) **Starea formularului este un signal**, nu un Observable -- validarile, erorile si starea de submitting sunt accesibile ca signals (`loginForm.email().errors()`, `loginForm().submitting()`). (2) **Declaratie declarativa a validarilor** -- validatorii se aplica intr-o functie de configurare (`required(f.email)`, `minLength(f.password, 8)`) in loc de array-uri in `FormGroup`. (3) **Model-driven** -- formularul se leaga de un `signal()` care contine modelul de date, eliminand sincronizarea manuala intre form si model. (4) **Submit gestionat** -- functia `submit()` gestioneaza automat starea de submitting si previne submit-uri multiple. (5) Se integreaza nativ cu `effect()` si `computed()`, eliminand necesitatea `valueChanges` Observable. Reactive Forms raman stabile si suportate, dar Signal Forms reprezinta directia viitoare.

### 12. Ce inseamna "zoneless" in Angular 20+ si de ce este important?

**Raspuns:** Incepand cu Angular 20, `provideZonelessChangeDetection()` inlocuieste Zone.js ca mecanism de change detection, iar din Angular 21 este **default pentru aplicatii noi**. Zone.js era o librarie care "patch-uia" toate API-urile asincrone (setTimeout, Promise, addEventListener) pentru a notifica Angular sa ruleze change detection. Problemele Zone.js: (1) overhead de performanta (~100KB + monkey-patching), (2) incompatibilitate cu unele librarii si `async/await` nativ, (3) change detection excesiv (rula la ORICE eveniment async, nu doar cele relevante). In modul zoneless, Angular se bazeaza pe **signals** si pe **notificari explicite** (events din template, `markForCheck()`) pentru a sti cand sa verifice componente. Rezultatul: bundle mai mic, performanta mai buna, si change detection predictibil. Migrarea necesita ca toate componentele sa foloseasca signals sau OnPush cu input-uri imutabile.

### 13. Ce componente noi ofera Angular Aria si de ce sunt relevante?

**Raspuns:** Angular Aria (Developer Preview in Angular 21) este o colectie de **componente headless accesibile** -- ofera comportament si accesibilitate (ARIA roles, keyboard navigation, focus management) fara a impune stiluri vizuale. Include: Accordion, Combobox, Grid, Listbox, Menu, Tabs, Toolbar si Tree. Sunt relevante deoarece: (1) rezolva problema accesibilitatii "by default" -- dezvoltatorii nu mai trebuie sa implementeze manual ARIA attributes si keyboard handlers, (2) sunt **headless** (fara stiluri), deci se integreaza in orice design system, spre deosebire de Angular Material care vine cu Material Design, (3) urmeaza pattern-ul APG (ARIA Authoring Practices Guide) al W3C. Sunt comparabile cu Headless UI (React) sau Radix Primitives, dar native pentru Angular si integrate cu signals.
