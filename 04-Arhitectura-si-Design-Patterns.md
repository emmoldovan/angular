# 4. Arhitectura si Design Patterns in Angular

## Cuprins
1. [Arhitectura modulara si structura proiectelor mari](#1-arhitectura-modulara-si-structura-proiectelor-mari)
2. [Micro-frontends cu Module Federation / Native Federation](#2-micro-frontends-cu-module-federation--native-federation)
3. [Smart vs Dumb components pattern](#3-smart-vs-dumb-components-pattern)
4. [Facade pattern pentru state management](#4-facade-pattern-pentru-state-management)
5. [State management options](#5-state-management-options)
6. [Monorepo cu Nx](#6-monorepo-cu-nx)
7. [Design patterns aplicate in Angular](#7-design-patterns-aplicate-in-angular)
8. [Communication patterns intre componente](#8-communication-patterns-intre-componente)
9. [Intrebari frecvente de interviu](#9-intrebari-frecvente-de-interviu)

---

## 1. Arhitectura modulara si structura proiectelor mari

### Feature Modules vs Shared Modules vs Core Module

In aplicatiile Angular de dimensiuni mari, organizarea codului in module specializate este esentiala pentru mentinerea scalabilitatii si a claritatii arhitecturale.

**Core Module** - contine servicii singleton, guards, interceptors, si componente folosite o singura data (header, footer, layout). Se importa **doar** in `AppModule`.

```typescript
// core/core.module.ts
@NgModule({
  declarations: [HeaderComponent, FooterComponent, NotFoundComponent],
  imports: [CommonModule, RouterModule],
  exports: [HeaderComponent, FooterComponent],
})
export class CoreModule {
  // Prevenim importul multiplu - CoreModule trebuie sa fie singleton
  constructor(@Optional() @SkipSelf() parentModule: CoreModule) {
    if (parentModule) {
      throw new Error(
        'CoreModule este deja incarcat. Importa-l doar in AppModule!'
      );
    }
  }
}
```

**Shared Module** - contine componente, directive si pipes reutilizabile. Se importa in orice feature module care are nevoie.

```typescript
// shared/shared.module.ts
@NgModule({
  declarations: [
    LoadingSpinnerComponent,
    ConfirmDialogComponent,
    TruncatePipe,
    HighlightDirective,
  ],
  imports: [CommonModule, FormsModule, ReactiveFormsModule, MaterialModule],
  exports: [
    // Exportam tot ce trebuie reutilizat
    CommonModule,
    FormsModule,
    ReactiveFormsModule,
    MaterialModule,
    LoadingSpinnerComponent,
    ConfirmDialogComponent,
    TruncatePipe,
    HighlightDirective,
  ],
})
export class SharedModule {}
```

**Feature Module** - contine totul legat de o functionalitate specifica. Se incarca lazy cand este posibil.

```typescript
// features/orders/orders.module.ts
@NgModule({
  declarations: [
    OrderListComponent,
    OrderDetailComponent,
    OrderFormComponent,
  ],
  imports: [
    CommonModule,
    SharedModule,
    OrdersRoutingModule, // routing propriu feature-ului
  ],
})
export class OrdersModule {}
```

### Folder structure conventions (folder-by-feature)

Structura recomandata pentru un proiect Angular enterprise:

```
src/
  app/
    core/                          # Servicii singleton, guards, interceptors
      guards/
        auth.guard.ts
        role.guard.ts
      interceptors/
        auth.interceptor.ts
        error.interceptor.ts
        loading.interceptor.ts
      services/
        auth.service.ts
        notification.service.ts
      layout/
        header/
          header.component.ts
        footer/
          footer.component.ts
        sidebar/
          sidebar.component.ts
      core.module.ts

    shared/                        # Componente, pipes, directive reutilizabile
      components/
        loading-spinner/
        confirm-dialog/
        data-table/
        pagination/
      directives/
        highlight.directive.ts
        click-outside.directive.ts
      pipes/
        truncate.pipe.ts
        date-format.pipe.ts
      models/
        user.model.ts
        api-response.model.ts
      utils/
        validators.ts
        date-helpers.ts
      shared.module.ts

    features/                      # Module de functionalitate (lazy loaded)
      dashboard/
        components/
          dashboard-widget/
          stats-card/
        pages/
          dashboard-page/
        services/
          dashboard.service.ts
        dashboard-routing.module.ts
        dashboard.module.ts

      orders/
        components/                # Componente specifice orders
          order-list/
          order-detail/
          order-form/
          order-status-badge/
        pages/                     # Page-level (smart) components
          orders-page/
          order-detail-page/
        services/
          orders.service.ts
          orders-facade.service.ts
        models/
          order.model.ts
        state/                     # State management specific feature-ului
          orders.actions.ts
          orders.reducer.ts
          orders.effects.ts
          orders.selectors.ts
        orders-routing.module.ts
        orders.module.ts

    app-routing.module.ts
    app.module.ts
    app.component.ts
```

### Barrel exports (index.ts)

Barrel exports simplifica importurile si creeaza un API public clar pentru fiecare modul:

```typescript
// shared/components/index.ts
export * from './loading-spinner/loading-spinner.component';
export * from './confirm-dialog/confirm-dialog.component';
export * from './data-table/data-table.component';
export * from './pagination/pagination.component';

// shared/pipes/index.ts
export * from './truncate.pipe';
export * from './date-format.pipe';

// shared/index.ts - barrel principal
export * from './components';
export * from './pipes';
export * from './directives';
export * from './models';
export * from './utils';
```

Utilizarea devine mult mai curata:

```typescript
// In loc de importuri lungi si specifice:
import { LoadingSpinnerComponent } from '../shared/components/loading-spinner/loading-spinner.component';
import { TruncatePipe } from '../shared/pipes/truncate.pipe';
import { User } from '../shared/models/user.model';

// Folosim barrel exports:
import { LoadingSpinnerComponent, TruncatePipe, User } from '@shared';
```

Configurarea alias-urilor in `tsconfig.json`:

```json
{
  "compilerOptions": {
    "paths": {
      "@core/*": ["src/app/core/*"],
      "@shared": ["src/app/shared/index.ts"],
      "@shared/*": ["src/app/shared/*"],
      "@features/*": ["src/app/features/*"],
      "@env": ["src/environments/environment"]
    }
  }
}
```

### Standalone components si trecerea de la NgModules

Incepand cu Angular 14+, standalone components elimina necesitatea declararii in NgModules. In Angular 17+, standalone este implicit `true`.

```typescript
// Standalone component - nu necesita un NgModule
@Component({
  selector: 'app-user-card',
  standalone: true,
  imports: [
    CommonModule,          // pentru *ngIf, *ngFor, etc.
    RouterLink,            // pentru routerLink
    MatCardModule,         // Material modules direct
    DateFormatPipe,        // pipe standalone
  ],
  template: `
    <mat-card>
      <mat-card-header>
        <mat-card-title>{{ user().name }}</mat-card-title>
        <mat-card-subtitle>{{ user().email }}</mat-card-subtitle>
      </mat-card-header>
      <mat-card-content>
        <p>Membru din: {{ user().createdAt | dateFormat }}</p>
      </mat-card-content>
      <mat-card-actions>
        <a [routerLink]="['/users', user().id]">Vezi profil</a>
      </mat-card-actions>
    </mat-card>
  `,
})
export class UserCardComponent {
  user = input.required<User>();
}

// Standalone pipe
@Pipe({
  name: 'dateFormat',
  standalone: true,
})
export class DateFormatPipe implements PipeTransform {
  transform(value: Date | string, format = 'dd MMM yyyy'): string {
    return formatDate(value, format, 'ro');
  }
}
```

**Routing cu standalone components** - fara module de routing separate:

```typescript
// app.routes.ts
export const appRoutes: Routes = [
  {
    path: '',
    loadComponent: () =>
      import('./features/dashboard/dashboard-page.component')
        .then(m => m.DashboardPageComponent),
  },
  {
    path: 'orders',
    loadChildren: () =>
      import('./features/orders/orders.routes')
        .then(m => m.orderRoutes),
  },
  {
    path: 'users',
    loadChildren: () =>
      import('./features/users/users.routes')
        .then(m => m.userRoutes),
  },
];

// features/orders/orders.routes.ts
export const orderRoutes: Routes = [
  {
    path: '',
    component: OrdersPageComponent,
  },
  {
    path: ':id',
    component: OrderDetailPageComponent,
  },
];

// main.ts - bootstrap fara AppModule
bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(appRoutes, withPreloading(PreloadAllModules)),
    provideHttpClient(withInterceptors([authInterceptor, errorInterceptor])),
    provideAnimations(),
  ],
});
```

### Library approach pentru shared code

Cand codul partajat creste, se poate extrage intr-o librarie Angular (fie Nx library, fie Angular library standard):

```bash
# Creare librarie Angular cu ng-packagr
ng generate library @myorg/ui-components
ng generate library @myorg/shared-utils
ng generate library @myorg/auth
```

```typescript
// projects/ui-components/src/public-api.ts
export * from './lib/button/button.component';
export * from './lib/data-table/data-table.component';
export * from './lib/modal/modal.component';
export * from './lib/form-field/form-field.component';

// Consumul in aplicatie:
import { ButtonComponent, DataTableComponent } from '@myorg/ui-components';
```

Avantaje ale abordarii cu librarii:
- **Versioning independent** - fiecare librarie poate avea versiunea proprie
- **Build separat** - se recompileaza doar ce s-a schimbat
- **Reutilizare intre proiecte** - publicare in npm registry privat
- **Enforced boundaries** - dependentele sunt explicite prin `package.json`

---

## 2. Micro-frontends cu Module Federation / Native Federation

### Ce sunt micro-frontends si cand sa le folosesti

Micro-frontends extind conceptul de microservicii la frontend. Fiecare echipa detine o parte din aplicatie, cu deploy independent si technologie potential diferita.

**Cand sa folosesti micro-frontends:**
- Echipe mari (5+ echipe) care lucreaza pe aceeasi aplicatie
- Nevoia de deploy independent per echipa
- Parti ale aplicatiei cu cicluri de release diferite
- Migrare graduala de la o tehnologie la alta (ex: AngularJS la Angular)

**Cand sa NU folosesti:**
- Echipa mica (1-3 echipe) - overhead-ul nu justifica beneficiile
- Aplicatie cu UI puternic integrat (multe dependente intre parti)
- Proiect greenfield mic-mediu

### Webpack Module Federation basics

Module Federation (introdus in Webpack 5) permite incarcarea dinamica de cod din alte aplicatii la runtime.

**Shell (host) app** - aplicatia principala care incarca remote-urile:

```typescript
// webpack.config.js - Shell App
const { ModuleFederationPlugin } = require('webpack').container;

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'shell',
      remotes: {
        // Remote-urile sunt incarcate la runtime
        orders: 'orders@http://localhost:4201/remoteEntry.js',
        users: 'users@http://localhost:4202/remoteEntry.js',
        reports: 'reports@http://localhost:4203/remoteEntry.js',
      },
      shared: {
        '@angular/core': { singleton: true, strictVersion: true },
        '@angular/common': { singleton: true, strictVersion: true },
        '@angular/router': { singleton: true, strictVersion: true },
        rxjs: { singleton: true, strictVersion: true },
      },
    }),
  ],
};
```

**Remote app** - expune module catre shell:

```typescript
// webpack.config.js - Orders Remote
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'orders',
      filename: 'remoteEntry.js',
      exposes: {
        // Ce module sunt disponibile pentru shell
        './OrdersModule': './src/app/orders/orders.module.ts',
        './OrderWidget': './src/app/orders/widgets/order-widget.component.ts',
      },
      shared: {
        '@angular/core': { singleton: true, strictVersion: true },
        '@angular/common': { singleton: true, strictVersion: true },
        '@angular/router': { singleton: true, strictVersion: true },
        rxjs: { singleton: true, strictVersion: true },
      },
    }),
  ],
};
```

### @angular-architects/native-federation (pentru esbuild)

Incepand cu Angular 17+, esbuild este builder-ul implicit. `native-federation` ofera Module Federation fara Webpack:

```bash
# Instalare
ng add @angular-architects/native-federation --project shell --type host --port 4200
ng add @angular-architects/native-federation --project orders --type remote --port 4201
```

```typescript
// federation.config.js - Shell (host)
const { withNativeFederation, shareAll } = require('@angular-architects/native-federation/config');

module.exports = withNativeFederation({
  shared: {
    ...shareAll({
      singleton: true,
      strictVersion: true,
      requiredVersion: 'auto',
    }),
  },
  skip: ['rxjs/ajax', 'rxjs/fetch', 'rxjs/testing', 'rxjs/webSocket'],
});

// federation.config.js - Orders (remote)
module.exports = withNativeFederation({
  name: 'orders',
  exposes: {
    './routes': './src/app/orders/orders.routes.ts',
  },
  shared: {
    ...shareAll({
      singleton: true,
      strictVersion: true,
      requiredVersion: 'auto',
    }),
  },
  skip: ['rxjs/ajax', 'rxjs/fetch', 'rxjs/testing', 'rxjs/webSocket'],
});
```

### Shell app + remote apps pattern

```typescript
// Shell - routes cu lazy loading de remote-uri
// app.routes.ts
import { loadRemoteModule } from '@angular-architects/native-federation';

export const appRoutes: Routes = [
  {
    path: '',
    component: ShellLayoutComponent,
    children: [
      {
        path: 'dashboard',
        loadComponent: () =>
          import('./dashboard/dashboard.component')
            .then(m => m.DashboardComponent),
      },
      {
        path: 'orders',
        loadChildren: () =>
          loadRemoteModule('orders', './routes')
            .then(m => m.orderRoutes),
      },
      {
        path: 'users',
        loadChildren: () =>
          loadRemoteModule('users', './routes')
            .then(m => m.userRoutes),
      },
      {
        path: 'reports',
        loadChildren: () =>
          loadRemoteModule('reports', './routes')
            .then(m => m.reportRoutes),
      },
    ],
  },
];

// Manifest - configureaza remote-urile dinamic
// federation.manifest.json
{
  "orders": "http://localhost:4201/remoteEntry.json",
  "users": "http://localhost:4202/remoteEntry.json",
  "reports": "http://localhost:4203/remoteEntry.json"
}

// main.ts - Shell bootstrap cu manifest
import { initFederation } from '@angular-architects/native-federation';

initFederation('federation.manifest.json')
  .then(() => import('./bootstrap'))
  .catch(err => console.error(err));
```

### Shared dependencies management

Managementul dependentelor partajate este critic - incarci o singura instanta de Angular, nu una per remote:

```typescript
// Reguli pentru shared dependencies:
// 1. singleton: true - o singura instanta in tot ecosistemul
// 2. strictVersion: true - versiunile trebuie sa fie compatibile
// 3. requiredVersion: 'auto' - citeste din package.json

const sharedConfig = {
  '@angular/core': {
    singleton: true,
    strictVersion: true,
    requiredVersion: '^17.0.0',  // toate remote-urile pe Angular 17.x
  },
  '@angular/common': { singleton: true, strictVersion: true },
  '@angular/router': { singleton: true, strictVersion: true },
  '@angular/forms': { singleton: true, strictVersion: true },
  '@ngrx/store': { singleton: true, strictVersion: true },
  '@ngrx/effects': { singleton: true, strictVersion: true },
  rxjs: { singleton: true, strictVersion: true },

  // Librarii interne partajate
  '@myorg/shared-ui': { singleton: true, strictVersion: true },
  '@myorg/auth': { singleton: true, strictVersion: true },
};
```

### Trade-offs: complexitate vs autonomie de echipa

| Aspect | Avantaje | Dezavantaje |
|--------|----------|-------------|
| **Deploy** | Independent per echipa | Complexitate CI/CD crescuta |
| **Tehnologie** | Libertate in alegere | Posibila fragmentare a stack-ului |
| **Echipe** | Autonomie ridicata | Coordonare necesara pt shared deps |
| **Performance** | Incarcare doar ce e necesar | Overhead la runtime (remoteEntry.js) |
| **Testare** | Testare izolata per remote | Integration testing complex |
| **UX** | N/A | Risc de inconsistenta vizuala |
| **Debugging** | Module boundaries clare | Debugging cross-remote dificil |
| **Shared state** | Reducere coupling | Comunicarea intre remote-uri e complexa |

---

## 3. Smart vs Dumb components pattern

### Container (smart) components

Smart components (container components) se ocupa de:
- Interactiunea cu servicii si state management
- Logica de business
- Orchestrarea componentelor dumb
- Navigare si side-effects

### Presentational (dumb) components

Dumb components (presentational components) se ocupa de:
- Afisarea datelor primite prin `@Input()` / `input()`
- Emiterea de evenimente prin `@Output()` / `output()`
- Zero logica de business
- Zero dependente de servicii

### Beneficii

- **Testabilitate** - dumb components sunt usor de testat (doar input/output)
- **Reutilizabilitate** - dumb components pot fi folosite in contexte diferite
- **Separarea responsabilitatilor** - logica sta in smart, UI-ul in dumb
- **OnPush change detection** - dumb components functioneaza perfect cu OnPush

### Exemplu complet

```typescript
// ===== DUMB (Presentational) Component =====
// features/orders/components/order-list/order-list.component.ts
@Component({
  selector: 'app-order-list',
  standalone: true,
  imports: [DatePipe, CurrencyPipe],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="order-list">
      <div class="filters">
        <select (change)="filterChanged.emit($event.target.value)">
          <option value="all">Toate</option>
          <option value="pending">In asteptare</option>
          <option value="completed">Finalizate</option>
          <option value="cancelled">Anulate</option>
        </select>
      </div>

      <table>
        <thead>
          <tr>
            <th (click)="sortChanged.emit('id')">ID</th>
            <th (click)="sortChanged.emit('date')">Data</th>
            <th (click)="sortChanged.emit('total')">Total</th>
            <th>Status</th>
            <th>Actiuni</th>
          </tr>
        </thead>
        <tbody>
          @for (order of orders(); track order.id) {
            <tr [class.highlighted]="order.id === selectedOrderId()">
              <td>{{ order.id }}</td>
              <td>{{ order.date | date:'dd/MM/yyyy' }}</td>
              <td>{{ order.total | currency:'RON' }}</td>
              <td>
                <app-status-badge [status]="order.status" />
              </td>
              <td>
                <button (click)="orderSelected.emit(order)">
                  Detalii
                </button>
                <button (click)="orderDeleted.emit(order.id)">
                  Sterge
                </button>
              </td>
            </tr>
          } @empty {
            <tr>
              <td colspan="5">Nu exista comenzi.</td>
            </tr>
          }
        </tbody>
      </table>
    </div>
  `,
})
export class OrderListComponent {
  // Doar input-uri si output-uri - ZERO servicii injectate
  orders = input.required<Order[]>();
  selectedOrderId = input<number | null>(null);

  orderSelected = output<Order>();
  orderDeleted = output<number>();
  filterChanged = output<string>();
  sortChanged = output<string>();
}

// ===== Dumb component pentru status badge =====
@Component({
  selector: 'app-status-badge',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <span class="badge" [class]="'badge--' + status()">
      {{ statusLabel() }}
    </span>
  `,
})
export class StatusBadgeComponent {
  status = input.required<OrderStatus>();

  statusLabel = computed(() => {
    const labels: Record<OrderStatus, string> = {
      pending: 'In asteptare',
      processing: 'In procesare',
      completed: 'Finalizata',
      cancelled: 'Anulata',
    };
    return labels[this.status()];
  });
}

// ===== SMART (Container) Component =====
// features/orders/pages/orders-page/orders-page.component.ts
@Component({
  selector: 'app-orders-page',
  standalone: true,
  imports: [OrderListComponent, OrderDetailComponent, LoadingSpinnerComponent],
  template: `
    <div class="orders-page">
      <h1>Gestiune Comenzi</h1>

      @if (loading()) {
        <app-loading-spinner />
      } @else {
        <app-order-list
          [orders]="filteredOrders()"
          [selectedOrderId]="selectedOrderId()"
          (orderSelected)="onOrderSelected($event)"
          (orderDeleted)="onOrderDeleted($event)"
          (filterChanged)="onFilterChanged($event)"
          (sortChanged)="onSortChanged($event)"
        />
      }

      @if (selectedOrder()) {
        <app-order-detail
          [order]="selectedOrder()!"
          (close)="onDetailClosed()"
          (statusUpdated)="onStatusUpdated($event)"
        />
      }
    </div>
  `,
})
export class OrdersPageComponent implements OnInit {
  // Smart component injecteaza servicii si gestioneaza starea
  private readonly ordersFacade = inject(OrdersFacadeService);
  private readonly router = inject(Router);
  private readonly notification = inject(NotificationService);

  // State derivat din facade
  loading = this.ordersFacade.loading;
  orders = this.ordersFacade.orders;
  selectedOrder = this.ordersFacade.selectedOrder;
  selectedOrderId = computed(() => this.selectedOrder()?.id ?? null);

  // State local pentru filtrare/sortare
  currentFilter = signal<string>('all');
  currentSort = signal<string>('date');

  filteredOrders = computed(() => {
    let result = this.orders();
    const filter = this.currentFilter();
    if (filter !== 'all') {
      result = result.filter(o => o.status === filter);
    }
    return this.sortOrders(result, this.currentSort());
  });

  ngOnInit(): void {
    this.ordersFacade.loadOrders();
  }

  onOrderSelected(order: Order): void {
    this.ordersFacade.selectOrder(order.id);
  }

  onOrderDeleted(orderId: number): void {
    // Smart component gestioneaza confirmarea si side-effects
    if (confirm('Esti sigur ca vrei sa stergi comanda?')) {
      this.ordersFacade.deleteOrder(orderId);
      this.notification.success('Comanda stearsa cu succes');
    }
  }

  onFilterChanged(filter: string): void {
    this.currentFilter.set(filter);
  }

  onSortChanged(field: string): void {
    this.currentSort.set(field);
  }

  onDetailClosed(): void {
    this.ordersFacade.clearSelection();
  }

  onStatusUpdated(event: { orderId: number; status: OrderStatus }): void {
    this.ordersFacade.updateOrderStatus(event.orderId, event.status);
    this.notification.success('Status actualizat');
  }

  private sortOrders(orders: Order[], field: string): Order[] {
    return [...orders].sort((a, b) => {
      if (field === 'date') return new Date(b.date).getTime() - new Date(a.date).getTime();
      if (field === 'total') return b.total - a.total;
      return a.id - b.id;
    });
  }
}
```

---

## 4. Facade pattern pentru state management

### Conceptul de Service Facade

Facade pattern in Angular creeaza un strat de abstractie intre componente si logica complexa de state management. Componentele nu trebuie sa stie daca starea vine din NgRx, signals, BehaviorSubject sau HTTP direct.

**Beneficii:**
- Componentele raman simple si nu depind de implementarea starii
- Putem schimba state management-ul fara a modifica componentele
- Un singur punct de acces pentru fiecare feature
- Testare simplificata prin mock-uirea facade-ului

### Exemplu complet cu Facade

```typescript
// features/orders/models/order.model.ts
export interface Order {
  id: number;
  customerName: string;
  date: string;
  items: OrderItem[];
  total: number;
  status: OrderStatus;
}

export type OrderStatus = 'pending' | 'processing' | 'completed' | 'cancelled';

export interface OrderItem {
  productId: number;
  productName: string;
  quantity: number;
  price: number;
}

// ===== Facade Service =====
// features/orders/services/orders-facade.service.ts
@Injectable({ providedIn: 'root' })
export class OrdersFacadeService {
  private readonly http = inject(HttpClient);
  private readonly store = inject(Store); // NgRx store, ascuns de componente

  // ---- Selectori publici (ca signals) ----
  // Componentele consuma aceste signals fara sa stie de NgRx
  readonly orders = toSignal(this.store.select(selectAllOrders), {
    initialValue: [],
  });
  readonly selectedOrder = toSignal(this.store.select(selectSelectedOrder), {
    initialValue: null,
  });
  readonly loading = toSignal(this.store.select(selectOrdersLoading), {
    initialValue: false,
  });
  readonly error = toSignal(this.store.select(selectOrdersError), {
    initialValue: null,
  });
  readonly totalRevenue = toSignal(this.store.select(selectTotalRevenue), {
    initialValue: 0,
  });

  // Computed signals derivate
  readonly pendingOrders = computed(() =>
    this.orders().filter(o => o.status === 'pending')
  );
  readonly completedOrdersCount = computed(() =>
    this.orders().filter(o => o.status === 'completed').length
  );
  readonly hasOrders = computed(() => this.orders().length > 0);

  // ---- Actiuni publice (ascund dispatch-ul NgRx) ----
  loadOrders(): void {
    this.store.dispatch(OrdersActions.loadOrders());
  }

  selectOrder(orderId: number): void {
    this.store.dispatch(OrdersActions.selectOrder({ orderId }));
  }

  clearSelection(): void {
    this.store.dispatch(OrdersActions.clearSelection());
  }

  createOrder(order: Omit<Order, 'id'>): void {
    this.store.dispatch(OrdersActions.createOrder({ order }));
  }

  updateOrderStatus(orderId: number, status: OrderStatus): void {
    this.store.dispatch(OrdersActions.updateStatus({ orderId, status }));
  }

  deleteOrder(orderId: number): void {
    this.store.dispatch(OrdersActions.deleteOrder({ orderId }));
  }

  // Metode mai complexe care combina mai multe actiuni
  async duplicateOrder(orderId: number): Promise<void> {
    const original = this.orders().find(o => o.id === orderId);
    if (original) {
      const duplicate: Omit<Order, 'id'> = {
        ...original,
        date: new Date().toISOString(),
        status: 'pending',
      };
      this.createOrder(duplicate);
    }
  }

  exportOrders(format: 'csv' | 'pdf'): Observable<Blob> {
    return this.http.get(`/api/orders/export`, {
      params: { format },
      responseType: 'blob',
    });
  }
}

// ===== Alternativa: Facade FARA NgRx (cu signals) =====
// Aceeasi interfata publica, implementare complet diferita
@Injectable({ providedIn: 'root' })
export class OrdersFacadeService {
  private readonly http = inject(HttpClient);

  // State intern cu signals - ascuns de componente
  private readonly _orders = signal<Order[]>([]);
  private readonly _selectedOrderId = signal<number | null>(null);
  private readonly _loading = signal(false);
  private readonly _error = signal<string | null>(null);

  // ---- API public IDENTIC cu varianta NgRx ----
  readonly orders = this._orders.asReadonly();
  readonly loading = this._loading.asReadonly();
  readonly error = this._error.asReadonly();

  readonly selectedOrder = computed(() => {
    const id = this._selectedOrderId();
    return id ? this._orders().find(o => o.id === id) ?? null : null;
  });

  readonly totalRevenue = computed(() =>
    this._orders().reduce((sum, o) => sum + o.total, 0)
  );

  readonly pendingOrders = computed(() =>
    this._orders().filter(o => o.status === 'pending')
  );

  loadOrders(): void {
    this._loading.set(true);
    this._error.set(null);

    this.http.get<Order[]>('/api/orders').subscribe({
      next: (orders) => {
        this._orders.set(orders);
        this._loading.set(false);
      },
      error: (err) => {
        this._error.set(err.message);
        this._loading.set(false);
      },
    });
  }

  selectOrder(orderId: number): void {
    this._selectedOrderId.set(orderId);
  }

  clearSelection(): void {
    this._selectedOrderId.set(null);
  }

  createOrder(order: Omit<Order, 'id'>): void {
    this._loading.set(true);
    this.http.post<Order>('/api/orders', order).subscribe({
      next: (created) => {
        this._orders.update(orders => [...orders, created]);
        this._loading.set(false);
      },
      error: (err) => {
        this._error.set(err.message);
        this._loading.set(false);
      },
    });
  }

  updateOrderStatus(orderId: number, status: OrderStatus): void {
    this.http.patch<Order>(`/api/orders/${orderId}`, { status }).subscribe({
      next: (updated) => {
        this._orders.update(orders =>
          orders.map(o => o.id === orderId ? updated : o)
        );
      },
      error: (err) => this._error.set(err.message),
    });
  }

  deleteOrder(orderId: number): void {
    this.http.delete(`/api/orders/${orderId}`).subscribe({
      next: () => {
        this._orders.update(orders => orders.filter(o => o.id !== orderId));
        if (this._selectedOrderId() === orderId) {
          this._selectedOrderId.set(null);
        }
      },
      error: (err) => this._error.set(err.message),
    });
  }
}
```

Punctul cheie: **componentele nu se schimba** cand migrezi de la o implementare la alta. Facade-ul izoleaza complet detaliile de implementare.

---

## 5. State management options

### 5.1 Simple services cu BehaviorSubject

Cea mai simpla abordare, potrivita pentru aplicatii mici-medii sau state local per feature.

```typescript
// services/cart.service.ts
export interface CartState {
  items: CartItem[];
  loading: boolean;
  error: string | null;
}

const initialState: CartState = {
  items: [],
  loading: false,
  error: null,
};

@Injectable({ providedIn: 'root' })
export class CartService {
  private readonly state$ = new BehaviorSubject<CartState>(initialState);

  // Selectori ca Observable-uri
  readonly items$ = this.state$.pipe(
    map(state => state.items),
    distinctUntilChanged(),
  );

  readonly totalPrice$ = this.state$.pipe(
    map(state => state.items.reduce(
      (sum, item) => sum + item.price * item.quantity, 0
    )),
    distinctUntilChanged(),
  );

  readonly itemCount$ = this.state$.pipe(
    map(state => state.items.reduce(
      (count, item) => count + item.quantity, 0
    )),
    distinctUntilChanged(),
  );

  readonly loading$ = this.state$.pipe(
    map(state => state.loading),
    distinctUntilChanged(),
  );

  // Helper privat pentru update imutabil
  private updateState(partial: Partial<CartState>): void {
    this.state$.next({ ...this.state$.value, ...partial });
  }

  addItem(product: Product, quantity = 1): void {
    const items = [...this.state$.value.items];
    const existingIndex = items.findIndex(i => i.productId === product.id);

    if (existingIndex >= 0) {
      items[existingIndex] = {
        ...items[existingIndex],
        quantity: items[existingIndex].quantity + quantity,
      };
    } else {
      items.push({
        productId: product.id,
        name: product.name,
        price: product.price,
        quantity,
      });
    }

    this.updateState({ items });
  }

  removeItem(productId: number): void {
    this.updateState({
      items: this.state$.value.items.filter(i => i.productId !== productId),
    });
  }

  updateQuantity(productId: number, quantity: number): void {
    if (quantity <= 0) {
      this.removeItem(productId);
      return;
    }
    this.updateState({
      items: this.state$.value.items.map(item =>
        item.productId === productId ? { ...item, quantity } : item
      ),
    });
  }

  clearCart(): void {
    this.updateState({ items: [] });
  }
}
```

### 5.2 NgRx Store (actions, reducers, effects, selectors)

NgRx este solutia de state management completa pentru aplicatii mari, bazata pe pattern-ul Redux (unidirectional data flow).

```typescript
// ===== ACTIONS =====
// state/orders/orders.actions.ts
import { createActionGroup, emptyProps, props } from '@ngrx/store';

export const OrdersActions = createActionGroup({
  source: 'Orders',
  events: {
    // Load
    'Load Orders': emptyProps(),
    'Load Orders Success': props<{ orders: Order[] }>(),
    'Load Orders Failure': props<{ error: string }>(),

    // CRUD
    'Create Order': props<{ order: Omit<Order, 'id'> }>(),
    'Create Order Success': props<{ order: Order }>(),
    'Create Order Failure': props<{ error: string }>(),

    'Update Status': props<{ orderId: number; status: OrderStatus }>(),
    'Update Status Success': props<{ order: Order }>(),
    'Update Status Failure': props<{ error: string }>(),

    'Delete Order': props<{ orderId: number }>(),
    'Delete Order Success': props<{ orderId: number }>(),
    'Delete Order Failure': props<{ error: string }>(),

    // Selection
    'Select Order': props<{ orderId: number }>(),
    'Clear Selection': emptyProps(),
  },
});

// ===== REDUCER =====
// state/orders/orders.reducer.ts
import { createReducer, on } from '@ngrx/store';
import { EntityState, EntityAdapter, createEntityAdapter } from '@ngrx/entity';

export interface OrdersState extends EntityState<Order> {
  selectedOrderId: number | null;
  loading: boolean;
  error: string | null;
}

export const ordersAdapter: EntityAdapter<Order> = createEntityAdapter<Order>({
  selectId: (order) => order.id,
  sortComparer: (a, b) => new Date(b.date).getTime() - new Date(a.date).getTime(),
});

const initialState: OrdersState = ordersAdapter.getInitialState({
  selectedOrderId: null,
  loading: false,
  error: null,
});

export const ordersReducer = createReducer(
  initialState,

  // Load
  on(OrdersActions.loadOrders, (state) => ({
    ...state,
    loading: true,
    error: null,
  })),
  on(OrdersActions.loadOrdersSuccess, (state, { orders }) =>
    ordersAdapter.setAll(orders, { ...state, loading: false })
  ),
  on(OrdersActions.loadOrdersFailure, (state, { error }) => ({
    ...state,
    loading: false,
    error,
  })),

  // Create
  on(OrdersActions.createOrderSuccess, (state, { order }) =>
    ordersAdapter.addOne(order, state)
  ),

  // Update
  on(OrdersActions.updateStatusSuccess, (state, { order }) =>
    ordersAdapter.updateOne({ id: order.id, changes: order }, state)
  ),

  // Delete
  on(OrdersActions.deleteOrderSuccess, (state, { orderId }) =>
    ordersAdapter.removeOne(orderId, {
      ...state,
      selectedOrderId: state.selectedOrderId === orderId
        ? null
        : state.selectedOrderId,
    })
  ),

  // Selection
  on(OrdersActions.selectOrder, (state, { orderId }) => ({
    ...state,
    selectedOrderId: orderId,
  })),
  on(OrdersActions.clearSelection, (state) => ({
    ...state,
    selectedOrderId: null,
  })),
);

// ===== SELECTORS =====
// state/orders/orders.selectors.ts
import { createFeatureSelector, createSelector } from '@ngrx/store';

const selectOrdersState = createFeatureSelector<OrdersState>('orders');

const { selectAll, selectEntities, selectTotal } =
  ordersAdapter.getSelectors(selectOrdersState);

export const selectAllOrders = selectAll;
export const selectOrderEntities = selectEntities;
export const selectOrdersCount = selectTotal;

export const selectOrdersLoading = createSelector(
  selectOrdersState,
  (state) => state.loading,
);

export const selectOrdersError = createSelector(
  selectOrdersState,
  (state) => state.error,
);

export const selectSelectedOrderId = createSelector(
  selectOrdersState,
  (state) => state.selectedOrderId,
);

export const selectSelectedOrder = createSelector(
  selectOrderEntities,
  selectSelectedOrderId,
  (entities, selectedId) => selectedId ? entities[selectedId] ?? null : null,
);

export const selectOrdersByStatus = (status: OrderStatus) =>
  createSelector(selectAllOrders, (orders) =>
    orders.filter(o => o.status === status)
  );

export const selectTotalRevenue = createSelector(
  selectAllOrders,
  (orders) => orders
    .filter(o => o.status === 'completed')
    .reduce((sum, o) => sum + o.total, 0),
);

// ===== EFFECTS =====
// state/orders/orders.effects.ts
import { Actions, createEffect, ofType } from '@ngrx/effects';

@Injectable()
export class OrdersEffects {
  private readonly actions$ = inject(Actions);
  private readonly ordersApi = inject(OrdersApiService);
  private readonly notification = inject(NotificationService);

  loadOrders$ = createEffect(() =>
    this.actions$.pipe(
      ofType(OrdersActions.loadOrders),
      switchMap(() =>
        this.ordersApi.getAll().pipe(
          map(orders => OrdersActions.loadOrdersSuccess({ orders })),
          catchError(error =>
            of(OrdersActions.loadOrdersFailure({ error: error.message }))
          ),
        )
      ),
    )
  );

  createOrder$ = createEffect(() =>
    this.actions$.pipe(
      ofType(OrdersActions.createOrder),
      exhaustMap(({ order }) =>
        this.ordersApi.create(order).pipe(
          map(created => OrdersActions.createOrderSuccess({ order: created })),
          catchError(error =>
            of(OrdersActions.createOrderFailure({ error: error.message }))
          ),
        )
      ),
    )
  );

  updateStatus$ = createEffect(() =>
    this.actions$.pipe(
      ofType(OrdersActions.updateStatus),
      concatMap(({ orderId, status }) =>
        this.ordersApi.updateStatus(orderId, status).pipe(
          map(order => OrdersActions.updateStatusSuccess({ order })),
          catchError(error =>
            of(OrdersActions.updateStatusFailure({ error: error.message }))
          ),
        )
      ),
    )
  );

  deleteOrder$ = createEffect(() =>
    this.actions$.pipe(
      ofType(OrdersActions.deleteOrder),
      mergeMap(({ orderId }) =>
        this.ordersApi.delete(orderId).pipe(
          map(() => OrdersActions.deleteOrderSuccess({ orderId })),
          catchError(error =>
            of(OrdersActions.deleteOrderFailure({ error: error.message }))
          ),
        )
      ),
    )
  );

  // Effect doar pentru side-effects (fara actiune rezultat)
  showSuccessNotification$ = createEffect(() =>
    this.actions$.pipe(
      ofType(
        OrdersActions.createOrderSuccess,
        OrdersActions.updateStatusSuccess,
        OrdersActions.deleteOrderSuccess,
      ),
      tap((action) => {
        const messages: Record<string, string> = {
          '[Orders] Create Order Success': 'Comanda creata cu succes',
          '[Orders] Update Status Success': 'Status actualizat',
          '[Orders] Delete Order Success': 'Comanda stearsa',
        };
        this.notification.success(messages[action.type] ?? 'Operatie reusita');
      }),
    ),
    { dispatch: false },
  );
}

// ===== REGISTRATION =====
// main.ts sau app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideStore({ orders: ordersReducer }),
    provideEffects(OrdersEffects),
    provideStoreDevtools({
      maxAge: 25,
      logOnly: !isDevMode(),
    }),
  ],
};
```

### 5.3 NgRx SignalStore (abordarea moderna cu signals)

NgRx SignalStore este alternativa moderna, construita pe Angular signals. Mai putin boilerplate, integrare nativa cu signals.

```typescript
// state/orders/orders.store.ts
import {
  signalStore,
  withState,
  withComputed,
  withMethods,
  withHooks,
  patchState,
} from '@ngrx/signals';
import { withEntities, setAllEntities, addEntity, updateEntity, removeEntity } from '@ngrx/signals/entities';
import { rxMethod } from '@ngrx/signals/rxjs-interop';

// Definirea starii
type OrdersState = {
  selectedOrderId: number | null;
  loading: boolean;
  error: string | null;
  filter: OrderStatus | 'all';
};

const initialState: OrdersState = {
  selectedOrderId: null,
  loading: false,
  error: null,
  filter: 'all',
};

export const OrdersStore = signalStore(
  // Facem store-ul disponibil global (providedIn: 'root')
  { providedIn: 'root' },

  // State de baza
  withState(initialState),

  // Entity management (inlocuieste @ngrx/entity)
  withEntities<Order>(),

  // Computed signals (inlocuieste selectors)
  withComputed((store) => ({
    selectedOrder: computed(() => {
      const id = store.selectedOrderId();
      return store.entities().find(o => o.id === id) ?? null;
    }),

    filteredOrders: computed(() => {
      const filter = store.filter();
      const orders = store.entities();
      return filter === 'all'
        ? orders
        : orders.filter(o => o.status === filter);
    }),

    totalRevenue: computed(() =>
      store.entities()
        .filter(o => o.status === 'completed')
        .reduce((sum, o) => sum + o.total, 0)
    ),

    pendingCount: computed(() =>
      store.entities().filter(o => o.status === 'pending').length
    ),

    hasOrders: computed(() => store.entities().length > 0),
  })),

  // Metode (inlocuieste actions + effects + reducer)
  withMethods((store, ordersApi = inject(OrdersApiService)) => ({
    // Metoda sincrona simpla
    selectOrder(orderId: number): void {
      patchState(store, { selectedOrderId: orderId });
    },

    clearSelection(): void {
      patchState(store, { selectedOrderId: null });
    },

    setFilter(filter: OrderStatus | 'all'): void {
      patchState(store, { filter });
    },

    // Metoda asincrona cu rxMethod (echivalent NgRx Effects)
    loadOrders: rxMethod<void>(
      pipe(
        tap(() => patchState(store, { loading: true, error: null })),
        switchMap(() =>
          ordersApi.getAll().pipe(
            tapResponse({
              next: (orders) => {
                patchState(store, setAllEntities(orders));
                patchState(store, { loading: false });
              },
              error: (err: HttpErrorResponse) => {
                patchState(store, {
                  loading: false,
                  error: err.message,
                });
              },
            }),
          )
        ),
      )
    ),

    // CRUD operations
    createOrder: rxMethod<Omit<Order, 'id'>>(
      pipe(
        tap(() => patchState(store, { loading: true })),
        exhaustMap((order) =>
          ordersApi.create(order).pipe(
            tapResponse({
              next: (created) => {
                patchState(store, addEntity(created));
                patchState(store, { loading: false });
              },
              error: (err: HttpErrorResponse) => {
                patchState(store, { loading: false, error: err.message });
              },
            }),
          )
        ),
      )
    ),

    updateStatus: rxMethod<{ orderId: number; status: OrderStatus }>(
      pipe(
        concatMap(({ orderId, status }) =>
          ordersApi.updateStatus(orderId, status).pipe(
            tapResponse({
              next: (updated) => {
                patchState(store, updateEntity({
                  id: updated.id,
                  changes: updated,
                }));
              },
              error: (err: HttpErrorResponse) => {
                patchState(store, { error: err.message });
              },
            }),
          )
        ),
      )
    ),

    deleteOrder: rxMethod<number>(
      pipe(
        mergeMap((orderId) =>
          ordersApi.delete(orderId).pipe(
            tapResponse({
              next: () => {
                patchState(store, removeEntity(orderId));
                if (store.selectedOrderId() === orderId) {
                  patchState(store, { selectedOrderId: null });
                }
              },
              error: (err: HttpErrorResponse) => {
                patchState(store, { error: err.message });
              },
            }),
          )
        ),
      )
    ),
  })),

  // Lifecycle hooks
  withHooks({
    onInit(store) {
      // Incarcare automata la initializare
      store.loadOrders();
    },
    onDestroy(store) {
      console.log('OrdersStore destroyed');
    },
  }),
);

// ===== Utilizare in componenta =====
@Component({
  selector: 'app-orders-page',
  standalone: true,
  imports: [OrderListComponent],
  template: `
    @if (store.loading()) {
      <app-loading-spinner />
    } @else {
      <app-order-list
        [orders]="store.filteredOrders()"
        [selectedOrderId]="store.selectedOrderId()"
        (orderSelected)="store.selectOrder($event.id)"
        (orderDeleted)="store.deleteOrder($event)"
        (filterChanged)="store.setFilter($event)"
      />
    }
  `,
})
export class OrdersPageComponent {
  // Injectare directa a store-ului
  readonly store = inject(OrdersStore);
}
```

### Cand sa folosesti fiecare abordare

| Criteriu | BehaviorSubject | NgRx Store | NgRx SignalStore |
|----------|----------------|------------|------------------|
| **Dimensiune proiect** | Mic-mediu | Mare-enterprise | Mediu-mare |
| **Complexitate state** | Simpla | Complexa, multe entitati | Medie-complexa |
| **Echipa** | 1-3 dev | 5+ dev | 2-5 dev |
| **Boilerplate** | Minimal | Mult (actions/reducers/effects) | Moderat |
| **DevTools** | Nu | Da (Redux DevTools) | Da (SignalStore DevTools) |
| **Time-travel debug** | Nu | Da | Partial |
| **Learning curve** | Joasa | Ridicata | Medie |
| **Predictibilitate** | Medie | Foarte ridicata | Ridicata |
| **Testabilitate** | Buna | Excelenta (componente izolate) | Foarte buna |
| **Angular version** | Orice | Orice | 17+ (signals) |

**Recomandare practica:**
- **Proiect mic, 1-2 dev:** BehaviorSubject services
- **Proiect mediu, 3-5 dev:** NgRx SignalStore sau Facade cu signals
- **Proiect enterprise, 5+ dev, state complex:** NgRx Store complet
- **Migrare graduala:** Incepe cu services, extrage in SignalStore, migreaza la NgRx daca e nevoie

---

## 6. Monorepo cu Nx

### Nx workspace structure

Nx transforma un singur repository in workspace structurat cu multiple aplicatii si librarii, cu tooling avansat pentru build, test si CI.

```bash
# Creare workspace Nx
npx create-nx-workspace@latest myorg --preset=angular-monorepo

# Structura generata:
myorg/
  apps/
    web-app/                     # Aplicatia principala Angular
      src/
      project.json
    web-app-e2e/                 # E2E tests
    admin-portal/                # A doua aplicatie Angular
    mobile-app/                  # Aplicatie NativeScript/Ionic

  libs/                          # Librarii partajate
    shared/
      ui/                        # Componente UI reutilizabile
        src/
          lib/
            button/
            modal/
            data-table/
          index.ts               # Public API
        project.json
      util/                      # Utilitare pure (fara Angular dependency)
        src/
          lib/
            date-helpers.ts
            validators.ts
          index.ts
        project.json
      models/                    # Interfete si tipuri partajate
        src/
          lib/
            user.model.ts
            order.model.ts
          index.ts
        project.json

    features/
      orders/
        data-access/             # Servicii, state management pentru orders
          src/
            lib/
              orders-api.service.ts
              orders.store.ts
            index.ts
          project.json
        feature-list/            # Smart component: lista de comenzi
          src/
            lib/
              orders-list.component.ts
            index.ts
          project.json
        feature-detail/          # Smart component: detaliu comanda
        ui/                      # Dumb components specifice orders

      users/
        data-access/
        feature-list/
        feature-detail/
        ui/

    auth/
      data-access/               # Auth service, token management
      feature-login/             # Login page component
      feature-register/          # Register page component

  nx.json                        # Configurare Nx
  tsconfig.base.json             # TypeScript paths pentru librarii
```

### Libraries: feature, data-access, ui, util

Nx recomanda o clasificare clara a librariilor:

```
# Tipuri de librarii Nx:

feature/     - Smart components, pages, routing
               Importa: data-access, ui, util
               Exemplu: feature-orders-list, feature-user-profile

data-access/ - Servicii, state management, API calls
               Importa: util, models
               Exemplu: data-access-orders, data-access-auth

ui/          - Dumb/presentational components, pipes, directives
               Importa: util (NU data-access sau feature)
               Exemplu: ui-buttons, ui-forms, ui-layout

util/        - Helper functions, constante, validators
               Importa: NIMIC (sau alte util)
               Exemplu: util-dates, util-formatting

models/      - Interfete TypeScript, enums, types
               Importa: NIMIC
               Exemplu: models-order, models-user
```

```bash
# Generare librarii cu Nx
nx generate @nx/angular:library shared-ui --directory=libs/shared/ui --standalone
nx generate @nx/angular:library orders-data-access --directory=libs/features/orders/data-access
nx generate @nx/angular:library orders-feature-list --directory=libs/features/orders/feature-list

# tsconfig.base.json - path aliases automate
{
  "compilerOptions": {
    "paths": {
      "@myorg/shared/ui": ["libs/shared/ui/src/index.ts"],
      "@myorg/shared/util": ["libs/shared/util/src/index.ts"],
      "@myorg/shared/models": ["libs/shared/models/src/index.ts"],
      "@myorg/orders/data-access": ["libs/features/orders/data-access/src/index.ts"],
      "@myorg/orders/feature-list": ["libs/features/orders/feature-list/src/index.ts"],
      "@myorg/auth/data-access": ["libs/auth/data-access/src/index.ts"]
    }
  }
}
```

### Dependency constraints si module boundaries

Nx enforce-uiaza reguli de dependenta intre librarii prin ESLint:

```json
// nx.json sau project.json - taguri pe librarii
// libs/shared/ui/project.json
{
  "name": "shared-ui",
  "tags": ["scope:shared", "type:ui"]
}

// libs/features/orders/data-access/project.json
{
  "name": "orders-data-access",
  "tags": ["scope:orders", "type:data-access"]
}

// libs/features/orders/feature-list/project.json
{
  "name": "orders-feature-list",
  "tags": ["scope:orders", "type:feature"]
}
```

```json
// .eslintrc.json - reguli de dependenta
{
  "rules": {
    "@nx/enforce-module-boundaries": [
      "error",
      {
        "depConstraints": [
          {
            "sourceTag": "type:feature",
            "onlyDependOnLibsWithTags": ["type:data-access", "type:ui", "type:util", "type:models"]
          },
          {
            "sourceTag": "type:data-access",
            "onlyDependOnLibsWithTags": ["type:util", "type:models"]
          },
          {
            "sourceTag": "type:ui",
            "onlyDependOnLibsWithTags": ["type:util", "type:models"]
          },
          {
            "sourceTag": "type:util",
            "onlyDependOnLibsWithTags": ["type:util", "type:models"]
          },
          {
            "sourceTag": "scope:orders",
            "onlyDependOnLibsWithTags": ["scope:orders", "scope:shared"]
          },
          {
            "sourceTag": "scope:users",
            "onlyDependOnLibsWithTags": ["scope:users", "scope:shared"]
          },
          {
            "sourceTag": "scope:shared",
            "onlyDependOnLibsWithTags": ["scope:shared"]
          }
        ]
      }
    ]
  }
}
```

Aceasta configurare previne:
- `ui` sa importeze din `data-access` sau `feature`
- `orders` sa importeze din `users` (fara a trece prin `shared`)
- Dependente circulare intre module

### Nx generators si executors

```bash
# Generators - creeaza cod conform conventiilor echipei
# Generator built-in
nx generate @nx/angular:component user-card --project=shared-ui --standalone

# Generator custom (workspace generator)
nx generate @nx/workspace:generator feature-module --directory=tools/generators

# Custom generator exemplu (tools/generators/feature-module/index.ts)
```

```typescript
// tools/generators/feature-module/index.ts
import { Tree, formatFiles, generateFiles, joinPathFragments } from '@nx/devkit';

interface FeatureModuleSchema {
  name: string;       // ex: "products"
  scope: string;      // ex: "features"
}

export default async function (tree: Tree, schema: FeatureModuleSchema) {
  const baseDir = `libs/${schema.scope}/${schema.name}`;

  // Genereaza data-access, feature, ui libraries dintr-o singura comanda
  const libraries = ['data-access', 'feature-list', 'feature-detail', 'ui'];

  for (const libType of libraries) {
    generateFiles(
      tree,
      joinPathFragments(__dirname, `./files/${libType}`),
      `${baseDir}/${libType}`,
      { name: schema.name, tmpl: '' },
    );
  }

  await formatFiles(tree);
}
```

```bash
# Executors - ruleaza tasks (build, test, lint, etc.)
# Built-in executors
nx build web-app                   # Build aplicatie
nx test shared-ui                  # Test librarie
nx lint orders-data-access         # Lint librarie
nx e2e web-app-e2e                 # E2E tests

# Custom executor in project.json
{
  "targets": {
    "build": {
      "executor": "@nx/angular:application",
      "options": { "outputPath": "dist/apps/web-app" }
    },
    "deploy": {
      "executor": "./tools/executors/deploy:deploy",
      "options": { "environment": "production" }
    }
  }
}
```

### Affected commands pentru CI optimization

Cel mai puternic feature Nx: ruleaza tasks doar pe proiectele afectate de modificari.

```bash
# Nx calculeaza dependency graph si ruleaza doar ce e afectat
# de modificarile din branch-ul curent vs main

nx affected -t build              # Build doar proiecte afectate
nx affected -t test               # Test doar proiecte afectate
nx affected -t lint               # Lint doar proiecte afectate
nx affected -t e2e                # E2E doar pe aplicatii afectate

# Vizualizare dependency graph
nx graph                          # Deschide graph interactiv in browser
nx affected:graph                 # Doar nodurile afectate

# In CI pipeline (GitHub Actions exemplu):
# .github/workflows/ci.yml
# - name: Run affected tests
#   run: npx nx affected -t test --base=origin/main --head=HEAD
# - name: Run affected builds
#   run: npx nx affected -t build --base=origin/main --head=HEAD

# Remote caching (Nx Cloud) - share build cache intre CI si dev
# nx.json
{
  "tasksRunnerOptions": {
    "default": {
      "runner": "nx-cloud",
      "options": {
        "accessToken": "xxx"
      }
    }
  }
}
```

**Exemplu impact:** Daca modifici `shared-ui`, Nx stie ca:
- `orders-feature-list` depinde de `shared-ui` -> se rebuilduieste si retesteaza
- `auth-data-access` NU depinde de `shared-ui` -> se sare peste
- Intr-un monorepo cu 50 librarii, CI ruleaza poate doar 5 in loc de 50

---

## 7. Design patterns aplicate in Angular

### Strategy pattern

Permite selectarea unui algoritm la runtime prin dependency injection.

```typescript
// Interfata strategiei
export interface PaymentStrategy {
  readonly name: string;
  processPayment(amount: number): Observable<PaymentResult>;
  validate(details: PaymentDetails): ValidationResult;
}

// Implementari concrete
@Injectable()
export class CreditCardPaymentStrategy implements PaymentStrategy {
  readonly name = 'Credit Card';
  private readonly http = inject(HttpClient);

  processPayment(amount: number): Observable<PaymentResult> {
    return this.http.post<PaymentResult>('/api/payments/card', { amount });
  }

  validate(details: PaymentDetails): ValidationResult {
    if (!details.cardNumber || details.cardNumber.length !== 16) {
      return { valid: false, error: 'Numar card invalid' };
    }
    return { valid: true };
  }
}

@Injectable()
export class PayPalPaymentStrategy implements PaymentStrategy {
  readonly name = 'PayPal';
  private readonly http = inject(HttpClient);

  processPayment(amount: number): Observable<PaymentResult> {
    return this.http.post<PaymentResult>('/api/payments/paypal', { amount });
  }

  validate(details: PaymentDetails): ValidationResult {
    if (!details.email) {
      return { valid: false, error: 'Email PayPal necesar' };
    }
    return { valid: true };
  }
}

@Injectable()
export class BankTransferPaymentStrategy implements PaymentStrategy {
  readonly name = 'Transfer bancar';
  private readonly http = inject(HttpClient);

  processPayment(amount: number): Observable<PaymentResult> {
    return this.http.post<PaymentResult>('/api/payments/bank', { amount });
  }

  validate(details: PaymentDetails): ValidationResult {
    if (!details.iban) {
      return { valid: false, error: 'IBAN necesar' };
    }
    return { valid: true };
  }
}

// Token de injectare pentru strategii
export const PAYMENT_STRATEGIES = new InjectionToken<PaymentStrategy[]>(
  'PaymentStrategies'
);

// Registration in providers
export const appConfig: ApplicationConfig = {
  providers: [
    {
      provide: PAYMENT_STRATEGIES,
      useClass: CreditCardPaymentStrategy,
      multi: true,
    },
    {
      provide: PAYMENT_STRATEGIES,
      useClass: PayPalPaymentStrategy,
      multi: true,
    },
    {
      provide: PAYMENT_STRATEGIES,
      useClass: BankTransferPaymentStrategy,
      multi: true,
    },
  ],
};

// Serviciu care foloseste strategia selectata
@Injectable({ providedIn: 'root' })
export class PaymentService {
  private readonly strategies = inject(PAYMENT_STRATEGIES);

  getAvailableStrategies(): PaymentStrategy[] {
    return this.strategies;
  }

  getStrategy(name: string): PaymentStrategy {
    const strategy = this.strategies.find(s => s.name === name);
    if (!strategy) {
      throw new Error(`Strategia de plata '${name}' nu exista`);
    }
    return strategy;
  }

  processPayment(strategyName: string, amount: number): Observable<PaymentResult> {
    const strategy = this.getStrategy(strategyName);
    return strategy.processPayment(amount);
  }
}
```

### Observer pattern (RxJS, EventEmitter)

Angular este construit pe Observer pattern prin RxJS. Orice `Observable`, `Subject`, `EventEmitter` implementeaza acest pattern.

```typescript
// Observer pattern cu RxJS Subject - event bus simplu
@Injectable({ providedIn: 'root' })
export class EventBusService {
  private readonly eventSubject = new Subject<AppEvent>();

  // Observable public - componentele se aboneaza
  readonly events$ = this.eventSubject.asObservable();

  // Emit event
  emit(event: AppEvent): void {
    this.eventSubject.next(event);
  }

  // Filtare pe tip de event
  on<T extends AppEvent['type']>(eventType: T): Observable<Extract<AppEvent, { type: T }>> {
    return this.events$.pipe(
      filter((event): event is Extract<AppEvent, { type: T }> =>
        event.type === eventType
      ),
    );
  }
}

// Tipuri de evenimente
type AppEvent =
  | { type: 'ORDER_CREATED'; payload: Order }
  | { type: 'USER_LOGGED_IN'; payload: User }
  | { type: 'NOTIFICATION'; payload: { message: string; severity: 'info' | 'error' } }
  | { type: 'THEME_CHANGED'; payload: 'light' | 'dark' };

// Utilizare
@Component({ /* ... */ })
export class HeaderComponent implements OnInit {
  private readonly eventBus = inject(EventBusService);
  private readonly destroyRef = inject(DestroyRef);

  ngOnInit(): void {
    // Observer: se aboneaza la evenimente
    this.eventBus.on('USER_LOGGED_IN').pipe(
      takeUntilDestroyed(this.destroyRef),
    ).subscribe(event => {
      console.log('User logat:', event.payload.name);
    });

    this.eventBus.on('THEME_CHANGED').pipe(
      takeUntilDestroyed(this.destroyRef),
    ).subscribe(event => {
      document.body.classList.toggle('dark-theme', event.payload === 'dark');
    });
  }
}

@Component({ /* ... */ })
export class LoginComponent {
  private readonly eventBus = inject(EventBusService);

  onLoginSuccess(user: User): void {
    // Observable: emite eveniment
    this.eventBus.emit({ type: 'USER_LOGGED_IN', payload: user });
  }
}
```

### Mediator pattern

Un serviciu actioneaza ca mediator intre componente care nu se cunosc direct.

```typescript
// Mediator service - coordoneaza interactiunile dintre componente
@Injectable({ providedIn: 'root' })
export class DashboardMediatorService {
  // State intern gestionat de mediator
  private readonly selectedDateRange = signal<DateRange>({
    start: startOfMonth(new Date()),
    end: new Date(),
  });

  private readonly selectedCategory = signal<string>('all');
  private readonly refreshTrigger = new Subject<void>();

  // Interfata publica - componentele interactioneaza DOAR cu mediatorul
  readonly dateRange = this.selectedDateRange.asReadonly();
  readonly category = this.selectedCategory.asReadonly();
  readonly refresh$ = this.refreshTrigger.asObservable();

  // Cand se schimba date range, mediatorul notifica toate componentele
  setDateRange(range: DateRange): void {
    this.selectedDateRange.set(range);
    // Mediatorul decide cine trebuie notificat
    this.refreshTrigger.next();
  }

  setCategory(category: string): void {
    this.selectedCategory.set(category);
    this.refreshTrigger.next();
  }

  // Componente diferite solicita date prin mediator
  requestExport(format: 'csv' | 'pdf'): void {
    // Mediatorul orchestreaza operatia complexa
    console.log(`Export ${format} for`, this.selectedDateRange(), this.selectedCategory());
  }
}

// Componenta A: Date Range Picker - comunica prin mediator
@Component({
  selector: 'app-date-range-picker',
  standalone: true,
  template: `<input type="date" (change)="onDateChange($event)">`,
})
export class DateRangePickerComponent {
  private readonly mediator = inject(DashboardMediatorService);

  onDateChange(event: Event): void {
    // Comunica prin mediator, nu direct cu alte componente
    this.mediator.setDateRange({ start: new Date(), end: new Date() });
  }
}

// Componenta B: Chart - reactioneaza la schimbarile din mediator
@Component({
  selector: 'app-sales-chart',
  standalone: true,
  template: `<canvas #chart></canvas>`,
})
export class SalesChartComponent implements OnInit {
  private readonly mediator = inject(DashboardMediatorService);
  private readonly destroyRef = inject(DestroyRef);

  ngOnInit(): void {
    // Reactioneaza la schimbari orchestrate de mediator
    this.mediator.refresh$.pipe(
      takeUntilDestroyed(this.destroyRef),
    ).subscribe(() => {
      const range = this.mediator.dateRange();
      const category = this.mediator.category();
      this.loadChartData(range, category);
    });
  }

  private loadChartData(range: DateRange, category: string): void {
    // Incarca date pe baza parametrilor din mediator
  }
}
```

### Singleton pattern (providedIn: 'root')

Angular DI creeaza singletons automat cand folosim `providedIn: 'root'`.

```typescript
// Singleton la nivel de aplicatie
@Injectable({ providedIn: 'root' })
export class AuthService {
  // O singura instanta in toata aplicatia
  private readonly currentUser = signal<User | null>(null);

  readonly isAuthenticated = computed(() => this.currentUser() !== null);
  readonly user = this.currentUser.asReadonly();

  login(credentials: Credentials): Observable<User> { /* ... */ }
  logout(): void { this.currentUser.set(null); }
}

// Singleton la nivel de feature (lazy loaded module)
// ATENTIE: daca modulul e lazy loaded, providedIn: 'root' tot creeaza singleton global
// Dar daca vrem singleton PER MODUL, folosim providers la nivel de route:

export const orderRoutes: Routes = [
  {
    path: '',
    providers: [
      // Aceste servicii sunt singleton DOAR in contextul orders
      OrdersLocalStateService,
      OrdersValidationService,
    ],
    children: [
      { path: '', component: OrderListComponent },
      { path: ':id', component: OrderDetailComponent },
    ],
  },
];

// Non-singleton: instanta noua per componenta
@Component({
  selector: 'app-form-builder',
  providers: [FormBuilderService], // Instanta noua per componenta
  template: `...`,
})
export class FormBuilderComponent {
  // Fiecare FormBuilderComponent are propriul FormBuilderService
  private readonly formBuilder = inject(FormBuilderService);
}
```

### Decorator pattern (Angular decorators)

Angular foloseste decoratori TypeScript extensiv. Putem crea si decoratori custom:

```typescript
// Angular built-in decorators:
// @Component, @Injectable, @Pipe, @Directive, @Input, @Output, etc.

// Custom property decorator - auto-persistare in localStorage
export function LocalStorage(key?: string) {
  return function (target: any, propertyName: string) {
    const storageKey = key || `${target.constructor.name}_${propertyName}`;

    const getter = function (this: any) {
      const value = localStorage.getItem(storageKey);
      return value ? JSON.parse(value) : undefined;
    };

    const setter = function (this: any, value: any) {
      if (value === undefined || value === null) {
        localStorage.removeItem(storageKey);
      } else {
        localStorage.setItem(storageKey, JSON.stringify(value));
      }
    };

    Object.defineProperty(target, propertyName, {
      get: getter,
      set: setter,
      enumerable: true,
      configurable: true,
    });
  };
}

// Utilizare
@Component({ /* ... */ })
export class SettingsComponent {
  @LocalStorage('app_theme') theme: string = 'light';
  @LocalStorage('app_language') language: string = 'ro';

  changeTheme(newTheme: string): void {
    this.theme = newTheme; // Auto-salvat in localStorage
  }
}

// Custom method decorator - logging
export function Log(message?: string) {
  return function (
    target: any,
    propertyKey: string,
    descriptor: PropertyDescriptor,
  ) {
    const original = descriptor.value;

    descriptor.value = function (...args: any[]) {
      const label = message || `${target.constructor.name}.${propertyKey}`;
      console.log(`[LOG] ${label} called with:`, args);
      const start = performance.now();

      const result = original.apply(this, args);

      // Suport pentru Promise si Observable
      if (result instanceof Observable) {
        return result.pipe(
          tap({
            next: (val) => console.log(`[LOG] ${label} emitted:`, val),
            error: (err) => console.error(`[LOG] ${label} error:`, err),
            complete: () => {
              const duration = performance.now() - start;
              console.log(`[LOG] ${label} completed in ${duration.toFixed(2)}ms`);
            },
          }),
        );
      }

      const duration = performance.now() - start;
      console.log(`[LOG] ${label} returned in ${duration.toFixed(2)}ms:`, result);
      return result;
    };

    return descriptor;
  };
}

// Utilizare
@Injectable({ providedIn: 'root' })
export class OrdersService {
  @Log('Incarcare comenzi')
  loadOrders(): Observable<Order[]> {
    return this.http.get<Order[]>('/api/orders');
  }

  @Log()
  calculateTotal(items: OrderItem[]): number {
    return items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  }
}
```

### Factory pattern (useFactory providers)

Factory pattern permite crearea de instante complexe in functie de conditii la runtime.

```typescript
// Factory simpla - configuratie bazata pe environment
export function loggerFactory(environment: Environment): LoggerService {
  if (environment.production) {
    return new RemoteLoggerService(); // Trimite logs la server
  }
  return new ConsoleLoggerService(); // Console.log in dev
}

export const appConfig: ApplicationConfig = {
  providers: [
    {
      provide: LoggerService,
      useFactory: () => loggerFactory(inject(ENVIRONMENT)),
    },
  ],
};

// Factory cu dependente injectate
export function httpServiceFactory(): HttpServiceBase {
  const platform = inject(PLATFORM_ID);
  const http = inject(HttpClient);
  const config = inject(APP_CONFIG);

  if (isPlatformBrowser(platform)) {
    // In browser: foloseste HttpClient cu interceptors
    return new BrowserHttpService(http, config);
  } else {
    // In SSR: foloseste un client optimizat pentru server
    return new ServerHttpService(http, config);
  }
}

// Factory pentru configurari dinamice
export const API_BASE_URL = new InjectionToken<string>('ApiBaseUrl');

export function apiBaseUrlFactory(): string {
  const config = inject(APP_CONFIG);
  const location = inject(DOCUMENT).location;

  // URL diferit bazat pe subdomain sau environment
  if (location.hostname.includes('staging')) {
    return config.stagingApiUrl;
  }
  if (location.hostname.includes('localhost')) {
    return config.devApiUrl;
  }
  return config.productionApiUrl;
}

// Factory pentru servicii complexe cu initializare asincrona
export function authServiceFactory(): AuthService {
  const http = inject(HttpClient);
  const router = inject(Router);
  const storage = inject(StorageService);
  const config = inject(AUTH_CONFIG);

  const service = new AuthService(http, router, storage);
  service.configure({
    tokenEndpoint: config.tokenUrl,
    refreshEndpoint: config.refreshUrl,
    clientId: config.clientId,
  });

  return service;
}

// Registration
export const appConfig: ApplicationConfig = {
  providers: [
    {
      provide: API_BASE_URL,
      useFactory: apiBaseUrlFactory,
    },
    {
      provide: HttpServiceBase,
      useFactory: httpServiceFactory,
    },
    {
      provide: AuthService,
      useFactory: authServiceFactory,
    },
  ],
};
```

---

## 8. Communication patterns intre componente

### @Input/@Output (parent-child)

Comunicarea standard intre parinte si copil:

```typescript
// Child component
@Component({
  selector: 'app-product-card',
  standalone: true,
  template: `
    <div class="card" [class.featured]="featured()">
      <h3>{{ product().name }}</h3>
      <p>{{ product().price | currency:'RON' }}</p>
      <button (click)="addToCart.emit(product())">Adauga in cos</button>
    </div>
  `,
})
export class ProductCardComponent {
  // Signal-based inputs (Angular 17+)
  product = input.required<Product>();
  featured = input(false);  // cu valoare default

  // Signal-based outputs (Angular 17+)
  addToCart = output<Product>();
}

// Model inputs (Angular 17.1+) - two-way binding
@Component({
  selector: 'app-rating',
  standalone: true,
  template: `
    @for (star of stars; track star) {
      <span
        [class.filled]="star <= value()"
        (click)="value.set(star)"
      >
        ★
      </span>
    }
  `,
})
export class RatingComponent {
  // model() suporta two-way binding cu [()]
  value = model(0);
  stars = [1, 2, 3, 4, 5];
}

// Parent component
@Component({
  selector: 'app-product-list',
  standalone: true,
  imports: [ProductCardComponent, RatingComponent],
  template: `
    @for (product of products(); track product.id) {
      <app-product-card
        [product]="product"
        [featured]="product.isFeatured"
        (addToCart)="onAddToCart($event)"
      />
      <!-- Two-way binding cu model() -->
      <app-rating [(value)]="product.rating" />
    }
  `,
})
export class ProductListComponent {
  products = input.required<Product[]>();

  onAddToCart(product: Product): void {
    console.log('Adaugat:', product.name);
  }
}
```

### Services cu Subject (comunicare intre siblings)

Cand doua componente nu sunt in relatie parinte-copil:

```typescript
// Shared service pentru comunicare
@Injectable({ providedIn: 'root' })
export class ProductSelectionService {
  private readonly selectedProduct$ = new BehaviorSubject<Product | null>(null);
  private readonly compareList$ = new BehaviorSubject<Product[]>([]);

  // Selectori publici
  readonly selected$ = this.selectedProduct$.asObservable();
  readonly compareList = toSignal(this.compareList$, { initialValue: [] });

  // Signal pentru selected (alternativa moderna)
  readonly selectedProduct = toSignal(this.selectedProduct$, { initialValue: null });

  select(product: Product): void {
    this.selectedProduct$.next(product);
  }

  addToCompare(product: Product): void {
    const current = this.compareList$.value;
    if (!current.find(p => p.id === product.id) && current.length < 4) {
      this.compareList$.next([...current, product]);
    }
  }

  removeFromCompare(productId: number): void {
    this.compareList$.next(
      this.compareList$.value.filter(p => p.id !== productId)
    );
  }
}

// Sibling A: Selecteaza un produs
@Component({
  selector: 'app-product-catalog',
  template: `
    @for (product of products(); track product.id) {
      <div (click)="selectionService.select(product)">
        {{ product.name }}
      </div>
    }
  `,
})
export class ProductCatalogComponent {
  readonly selectionService = inject(ProductSelectionService);
  products = input.required<Product[]>();
}

// Sibling B: Afiseaza produsul selectat
@Component({
  selector: 'app-product-preview',
  template: `
    @if (selected(); as product) {
      <div class="preview">
        <h2>{{ product.name }}</h2>
        <p>{{ product.description }}</p>
        <p>{{ product.price | currency:'RON' }}</p>
      </div>
    } @else {
      <p>Selecteaza un produs pentru preview</p>
    }
  `,
})
export class ProductPreviewComponent {
  private readonly selectionService = inject(ProductSelectionService);
  selected = this.selectionService.selectedProduct;
}
```

### NgRx/signals store (app-wide state)

Comunicarea prin store-ul global (vezi sectiunea 5 pentru detalii complete):

```typescript
// Orice componenta din aplicatie poate citi si modifica starea globala
@Component({
  template: `
    <span class="cart-badge">{{ cartCount() }}</span>
  `,
})
export class CartBadgeComponent {
  // Citeste din store-ul global
  private readonly store = inject(Store);
  cartCount = toSignal(this.store.select(selectCartItemCount), { initialValue: 0 });
}

@Component({
  template: `
    <button (click)="addToCart()">Adauga</button>
  `,
})
export class ProductDetailComponent {
  private readonly store = inject(Store);

  addToCart(): void {
    // Scrie in store-ul global
    this.store.dispatch(CartActions.addItem({ productId: 123, quantity: 1 }));
  }
}
```

### ViewChild (parent accessing child)

Parintele acceseaza direct proprietati si metode ale copilului:

```typescript
// Child component cu metode publice
@Component({
  selector: 'app-video-player',
  standalone: true,
  template: `
    <video #videoEl [src]="src()">
      <track kind="subtitles" />
    </video>
    <div class="controls">
      <span>{{ currentTime() | number:'1.0-0' }}s / {{ duration() }}s</span>
    </div>
  `,
})
export class VideoPlayerComponent {
  src = input.required<string>();

  private readonly videoEl = viewChild.required<ElementRef<HTMLVideoElement>>('videoEl');

  currentTime = signal(0);
  duration = signal(0);
  isPlaying = signal(false);

  play(): void {
    this.videoEl().nativeElement.play();
    this.isPlaying.set(true);
  }

  pause(): void {
    this.videoEl().nativeElement.pause();
    this.isPlaying.set(false);
  }

  seekTo(seconds: number): void {
    this.videoEl().nativeElement.currentTime = seconds;
  }
}

// Parent component acceseaza child-ul
@Component({
  selector: 'app-lesson-page',
  standalone: true,
  imports: [VideoPlayerComponent],
  template: `
    <app-video-player #player [src]="videoUrl()" />

    <div class="lesson-controls">
      <button (click)="player.play()">Play</button>
      <button (click)="player.pause()">Pause</button>
      <button (click)="skipToHighlight()">Salt la highlight</button>
      <p>Playing: {{ player.isPlaying() }}</p>
    </div>
  `,
})
export class LessonPageComponent {
  videoUrl = signal('/videos/lesson-1.mp4');

  // viewChild signal-based (Angular 17+)
  private readonly player = viewChild.required(VideoPlayerComponent);

  skipToHighlight(): void {
    this.player().seekTo(120); // Salt la secunda 120
  }
}
```

### Content projection

Permite componentelor sa accepte continut dinamic de la parinte:

```typescript
// Componenta cu content projection
@Component({
  selector: 'app-card',
  standalone: true,
  template: `
    <div class="card">
      <div class="card-header">
        <!-- Slot cu nume: header -->
        <ng-content select="[card-header]" />
      </div>

      <div class="card-body">
        <!-- Slot default (fara select) -->
        <ng-content />
      </div>

      <div class="card-footer">
        <!-- Slot cu nume: footer -->
        <ng-content select="[card-footer]" />
      </div>
    </div>
  `,
})
export class CardComponent {}

// Multi-slot content projection cu selectori CSS
@Component({
  selector: 'app-dialog',
  standalone: true,
  template: `
    <div class="dialog-overlay" (click)="close.emit()">
      <div class="dialog" (click)="$event.stopPropagation()">
        <div class="dialog-title">
          <ng-content select=".dialog-title" />
          <button (click)="close.emit()">X</button>
        </div>
        <div class="dialog-body">
          <ng-content />
        </div>
        <div class="dialog-actions">
          <ng-content select=".dialog-actions" />
        </div>
      </div>
    </div>
  `,
})
export class DialogComponent {
  close = output<void>();
}

// Utilizare
@Component({
  template: `
    <app-card>
      <h2 card-header>Titlu Card</h2>
      <p>Continut principal al card-ului</p>
      <p>Alt paragraf in body</p>
      <button card-footer (click)="save()">Salveaza</button>
    </app-card>

    <app-dialog (close)="closeDialog()">
      <span class="dialog-title">Confirmare stergere</span>

      <p>Esti sigur ca vrei sa stergi acest element?</p>
      <p>Aceasta actiune este ireversibila.</p>

      <div class="dialog-actions">
        <button (click)="closeDialog()">Anuleaza</button>
        <button (click)="confirmDelete()">Sterge</button>
      </div>
    </app-dialog>
  `,
})
export class ParentComponent {
  closeDialog(): void { /* ... */ }
  confirmDelete(): void { /* ... */ }
  save(): void { /* ... */ }
}

// Conditional content projection cu @ContentChild
@Component({
  selector: 'app-expandable-panel',
  standalone: true,
  template: `
    <div class="panel">
      <div class="panel-header" (click)="toggle()">
        <ng-content select="[panel-title]" />
        <span>{{ expanded() ? '▼' : '►' }}</span>
      </div>

      @if (expanded()) {
        <div class="panel-body">
          <ng-content />
        </div>
      }

      <!-- Arata custom footer daca exista, altfel default -->
      @if (hasCustomFooter()) {
        <ng-content select="[panel-footer]" />
      } @else {
        <div class="panel-footer-default">
          <small>Click header pentru expand/collapse</small>
        </div>
      }
    </div>
  `,
})
export class ExpandablePanelComponent {
  expanded = signal(false);

  // Detecteaza daca a fost proiectat continut in slot-ul footer
  private readonly customFooter = contentChild('panelFooter');
  hasCustomFooter = computed(() => !!this.customFooter());

  toggle(): void {
    this.expanded.update(v => !v);
  }
}
```

### Router params si query params

```typescript
// Transmitere de date prin router
// In componenta sursa:
@Component({
  template: `
    <!-- Route params -->
    <a [routerLink]="['/orders', order.id]">Detalii</a>

    <!-- Query params -->
    <a [routerLink]="['/orders']"
       [queryParams]="{ status: 'pending', page: 1 }">
      Comenzi in asteptare
    </a>

    <!-- Programatic -->
    <button (click)="goToOrder(order.id)">Deschide</button>
  `,
})
export class OrdersListComponent {
  private readonly router = inject(Router);

  goToOrder(id: number): void {
    this.router.navigate(['/orders', id], {
      queryParams: { tab: 'details' },
      // State invizibil in URL (nu apare in browser bar)
      state: { fromList: true, scrollPosition: 250 },
    });
  }
}

// In componenta destinatie - citirea parametrilor
// Abordarea moderna cu signals (Angular 16+):
@Component({
  selector: 'app-order-detail-page',
  standalone: true,
  template: `
    @if (order(); as order) {
      <h1>Comanda #{{ order.id }}</h1>
      <p>Status: {{ order.status }}</p>
    }
    <p>Tab activ: {{ activeTab() }}</p>
  `,
})
export class OrderDetailPageComponent implements OnInit {
  private readonly route = inject(ActivatedRoute);
  private readonly ordersService = inject(OrdersService);

  // Signal-based route params (Angular 16+ cu withComponentInputBinding())
  // Necesita: provideRouter(routes, withComponentInputBinding())
  id = input<string>();  // Automat din route param :id

  // Manual cu signals
  activeTab = toSignal(
    this.route.queryParamMap.pipe(
      map(params => params.get('tab') ?? 'details'),
    ),
    { initialValue: 'details' },
  );

  // Router state
  private readonly navigation = inject(Router).getCurrentNavigation();
  cameFromList = this.navigation?.extras?.state?.['fromList'] ?? false;

  order = toSignal(
    this.route.paramMap.pipe(
      map(params => Number(params.get('id'))),
      filter(id => !isNaN(id)),
      switchMap(id => this.ordersService.getById(id)),
    ),
  );
}

// Router config cu withComponentInputBinding()
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      appRoutes,
      withComponentInputBinding(), // Permite input() din route params
    ),
  ],
};
```

---

## 9. Intrebari frecvente de interviu

### 1. Care este diferenta dintre Core Module si Shared Module? Cand folosesti fiecare?

**Raspuns:** Core Module contine servicii singleton (AuthService, NotificationService), interceptors, guards si componente unice (Header, Footer). Se importa **o singura data** in AppModule. Shared Module contine componente, directive si pipes reutilizabile care se importa in **multiple feature modules**. Regula de baza: daca ceva are o singura instanta in aplicatie -> Core; daca se reutilizeaza in mai multe locuri -> Shared. Cu standalone components, Core Module devine mai putin necesar deoarece serviciile singleton cu `providedIn: 'root'` nu necesita modul, iar guards si interceptors se configureaza functional in `app.config.ts`. Shared Module ramane util ca barrel de re-export.

### 2. Explica pattern-ul Smart vs Dumb components. De ce este important?

**Raspuns:** Smart (Container) components injecteaza servicii, gestioneaza state-ul si logica de business. Dumb (Presentational) components primesc date exclusiv prin `input()` si comunica prin `output()`, fara dependente de servicii. Importanta: (1) **Testabilitate** - dumb components se testeaza trivial (seteaza input, verifica output), fara mock-uri de servicii; (2) **Reutilizabilitate** - un `OrderListComponent` dumb poate fi folosit in Dashboard, Admin Panel, Reports; (3) **OnPush performance** - dumb components functioneaza perfect cu `ChangeDetectionStrategy.OnPush`; (4) **Separarea responsabilitatilor** - echipa de UI lucreaza pe dumb components, echipa de business pe smart components. In practica, proportia ideala este ~80% dumb, ~20% smart.

### 3. Cand ai folosi NgRx Store vs NgRx SignalStore vs un simplu service cu BehaviorSubject?

**Raspuns:** **BehaviorSubject service** - pentru state simplu, feature-scoped, echipa mica. Exemple: cos de cumparaturi, preferinte utilizator, filtre locale. **NgRx SignalStore** - pentru state mediu-complex, cand vrei beneficiile unui framework de state management fara boilerplate-ul Redux complet. Integreaza nativ cu Angular signals, are entity management, si este mai usor de invatat. Recomandat pentru proiecte noi Angular 17+. **NgRx Store complet** - pentru aplicatii enterprise cu state global complex, multe entitati relationate, echipe mari care beneficiaza de predictibilitatea Redux (actions/reducers/effects). DevTools cu time-travel debugging sunt un avantaj major. Regula: incepe cu cel mai simplu, migreaza cand complexitatea o cere. Facade pattern permite migrarea fara sa modifici componentele.

### 4. Ce este Facade pattern si de ce il folosesti in Angular?

**Raspuns:** Facade pattern creeaza un strat de abstractie (un serviciu Angular) care expune o interfata simplificata catre componentele UI, ascunzand complexitatea state management-ului. De exemplu, un `OrdersFacadeService` expune `orders()`, `loading()`, `loadOrders()`, `deleteOrder()` - componentele nu stiu daca sub capota e NgRx, signals sau HTTP direct. **Beneficii:** (1) Decuplare completa - schimbi implementarea starii fara sa atingi componentele; (2) API curat - componentele nu au `store.dispatch(OrdersActions.loadOrders())`, ci `facade.loadOrders()`; (3) Testare - mock-uiezi un singur facade in teste; (4) Migrare graduala - poti trece de la BehaviorSubject la NgRx incremental. Este considerat un best practice in arhitecturi Angular enterprise.

### 5. Cum functioneaza Module Federation si cand ar trebui sa implementezi micro-frontends?

**Raspuns:** Module Federation (Webpack 5) / Native Federation (esbuild) permite incarcarea de cod JavaScript din alte aplicatii la runtime, fara sa fie nevoie de build-time integration. O aplicatie Shell (host) defineste remote-urile, iar fiecare remote expune module specifice. La navigare, shell-ul incarca dinamic remoteEntry.js si monteaza componentele remote-ului. **Cand sa folosesti:** echipe mari (5+) care au nevoie de deploy independent, aplicatii cu parti care au cicluri de release diferite, migrare graduala de la frameworkuri vechi. **Cand sa NU folosesti:** echipa mica, aplicatie cu UI tightly coupled, overhead-ul de coordonare a dependentelor partajate nu justifica autonomia. Shared dependencies (Angular core, RxJS) trebuie sa fie singleton si versiuni compatibile intre shell si remote-uri.

### 6. Cum structurezi un monorepo Nx pentru o aplicatie Angular enterprise?

**Raspuns:** Nx organizeaza codul in `apps/` (aplicatii deployabile) si `libs/` (librarii reutilizabile). Librariile au tipuri clare: **feature** (smart components, pages), **data-access** (servicii, state management), **ui** (dumb components, pipes), **util** (helper functions). Regulile de dependenta se enforce-uiaza prin `@nx/enforce-module-boundaries`: feature poate importa data-access si ui, dar ui NU poate importa feature sau data-access. Tags pe librarii (scope, type) definesc ce poate depinde de ce. **Affected commands** sunt killer feature: `nx affected -t test` ruleaza teste doar pe proiectele afectate de modificari, reducand dramatic timpul de CI. Combinat cu Nx Cloud remote caching, un CI care dura 30 min poate ajunge la 3 min.

### 7. Descrie cel putin 3 design patterns folosite frecvent in Angular si da exemple concrete.

**Raspuns:** (1) **Singleton** - `@Injectable({ providedIn: 'root' })` creeaza o singura instanta la nivel de aplicatie. AuthService, CartService sunt singletons. (2) **Strategy** - definesti o interfata (ex: `PaymentStrategy`) si injectezi implementari multiple (CardPayment, PayPal, BankTransfer) cu `InjectionToken` si `multi: true`. Componenta alege strategia la runtime. (3) **Observer** - RxJS este implementarea Observer pattern. Serviciile emit prin Subject, componentele se aboneaza cu subscribe sau toSignal. EventEmitter/@Output este tot Observer. (4) **Factory** - `useFactory` in providers creeaza instante conditionate: un LoggerService diferit in prod vs dev, sau un HttpService diferit pe browser vs SSR. (5) **Mediator** - un serviciu actioneaza ca hub central intre componente care nu se cunosc direct (ex: DashboardMediatorService coordoneaza DatePicker, Charts si Tables).

### 8. Care sunt toate modalitatile de comunicare intre componente in Angular? Cand folosesti fiecare?

**Raspuns:** (1) **@Input/@Output** - comunicare directa parinte-copil; cea mai simpla si preferata cand exista relatie directa. (2) **model()** - two-way binding (Angular 17.1+) pentru componente de tip form control. (3) **Services cu Subject** - comunicare intre siblings sau componente fara relatie directa; serviciul este mediator. (4) **NgRx/SignalStore** - state global, cand datele sunt necesare in locuri multiple si independente din aplicatie. (5) **viewChild()** - parintele acceseaza direct API-ul copilului (metode, proprietati); util pentru componente imperative (video player, canvas). (6) **Content projection** - parintele proiecteaza template in sloturile copilului; ideal pentru componente container generice (card, dialog, tabs). (7) **Router params/query params/state** - comunicare intre pagini prin URL; util pentru deep linking si share-ability. Regula: foloseste mecanismul cel mai simplu care rezolva problema. Input/Output acopera 70% din cazuri.

### 9. Ce este un barrel export (index.ts) si care sunt avantajele si dezavantajele?

**Raspuns:** Barrel export este un fisier `index.ts` care re-exporta simboluri din mai multe fisiere, oferind un singur punct de import. **Avantaje:** importuri curate (`import { A, B, C } from '@shared'` in loc de 3 importuri separate), API public clar (controlezi ce expui), refactoring usor (muti fisiere intern fara sa schimbi importurile externe). **Dezavantaje:** risc de circular dependencies daca nu esti atent, possible tree-shaking issues daca barrel-ul re-exporta totul (`export *`), si in proiecte foarte mari pot incetini compilarea TypeScript. **Best practice:** foloseste barrel exports pentru librarii si shared modules, dar evita re-exportul wildcard in barrel-uri de nivel inalt care agrupa sute de simboluri. In Nx, fiecare librarie are un `index.ts` ca public API - aceasta este practica standard enterprise.

### 10. Cum gestionezi comunicarea intre micro-frontends (remote-uri) intr-o arhitectura Module Federation?

**Raspuns:** Comunicarea intre remote-uri este intentionat limitata pentru a mentine decuplarea. Optiuni: (1) **Shared services** - servicii din librarii partajate (singleton shared) accesibile tuturor remote-urilor; de exemplu, AuthService partajat. (2) **Custom Events** (dispatchEvent/addEventListener pe window) - cel mai decuplat, functioneaza si cross-framework. (3) **Shared state store** - un NgRx store global in shell, accesibil remote-urilor. (4) **Router state** - comunicare prin URL params/query params. (5) **Event bus service** - un Subject global in shell, injectat in remote-uri. **Best practice:** minimizeaza comunicarea directa intre remote-uri. Daca doua remote-uri comunica mult, probabil ar trebui sa fie in acelasi remote. Shell-ul actioneaza ca mediator central.
