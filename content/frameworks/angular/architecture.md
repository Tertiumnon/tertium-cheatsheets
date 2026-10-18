# Angular Architecture

Modern Angular v20 architecture emphasizes standalone components, strong typing, signals for state management, and clear separation of concerns. This guide covers organizing large-scale Angular applications with scalability and maintainability in mind.

## Key Concepts

- **Standalone Components:** Self-contained components that don't require NgModules
- **Services:** Reusable business logic and data management
- **Dependency Injection (DI):** Angular's built-in IoC container for managing dependencies
- **Signals:** Reactive state management primitive (Angular v17+)
- **RxJS Observables:** Async data streams for reactive programming
- **Feature-Based Organization:** Organizing code by business features
- **Layered Architecture:** Separation of concerns with presentation, application, and infrastructure layers

## Project Structure (Feature-Based)

Organize by features, not technical layers:

```
src/
├── app/
│   ├── core/                         # Singleton services, guards, interceptors
│   │   ├── guards/
│   │   │   ├── auth.guard.ts
│   │   │   └── can-deactivate.guard.ts
│   │   ├── interceptors/
│   │   │   ├── error.interceptor.ts
│   │   │   └── auth.interceptor.ts
│   │   ├── services/
│   │   │   ├── auth.service.ts
│   │   │   ├── api.service.ts
│   │   │   └── storage.service.ts
│   │   └── core.config.ts            # CoreModule / DI config
│   ├── shared/                       # Reusable components, pipes, directives
│   │   ├── components/
│   │   │   ├── button/
│   │   │   │   ├── button.component.ts
│   │   │   │   ├── button.component.html
│   │   │   │   └── button.component.css
│   │   │   ├── modal/
│   │   │   ├── spinner/
│   │   │   └── index.ts              # Barrel export
│   │   ├── pipes/
│   │   │   ├── safe-html.pipe.ts
│   │   │   └── format-date.pipe.ts
│   │   ├── directives/
│   │   │   ├── highlight.directive.ts
│   │   │   └── click-outside.directive.ts
│   │   └── shared.config.ts          # SharedModule / DI config
│   ├── features/                     # Feature modules
│   │   ├── order/                    # Order Feature
│   │   │   ├── pages/                # Page components (smart components)
│   │   │   │   ├── order-list/
│   │   │   │   │   ├── order-list.component.ts
│   │   │   │   │   ├── order-list.component.html
│   │   │   │   │   └── order-list.component.css
│   │   │   │   └── order-detail/
│   │   │   ├── components/           # Presentational components
│   │   │   │   ├── order-card/
│   │   │   │   ├── order-form/
│   │   │   │   └── order-summary/
│   │   │   ├── services/             # Feature-specific services
│   │   │   │   ├── order.service.ts
│   │   │   │   └── order-facade.service.ts
│   │   │   ├── models/               # TypeScript interfaces/types
│   │   │   │   ├── order.model.ts
│   │   │   │   └── order-state.model.ts
│   │   │   ├── store/                # State management
│   │   │   │   ├── order.signal.ts   # or NgRx store
│   │   │   │   └── order.state.ts
│   │   │   ├── routes/               # Feature routes
│   │   │   │   └── order.routes.ts
│   │   │   └── order.config.ts       # Feature DI config
│   │   ├── customer/
│   │   │   ├── pages/
│   │   │   ├── components/
│   │   │   ├── services/
│   │   │   ├── models/
│   │   │   ├── store/
│   │   │   └── routes/
│   │   └── payment/
│   │       ├── pages/
│   │       ├── components/
│   │       └── ...
│   ├── app.component.ts              # Root component
│   ├── app.routes.ts                 # Root routes
│   └── app.config.ts                 # Root DI configuration
├── environments/                     # Environment configs
│   ├── environment.ts
│   └── environment.prod.ts
└── main.ts                           # Bootstrap
```

## Standalone Components (Angular v14+)

Modern Angular approach - no NgModules needed:

```typescript
// features/order/pages/order-list/order-list.component.ts
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterModule } from '@angular/router';
import { OrderService } from '../../services/order.service';
import { OrderCardComponent } from '../../components/order-card/order-card.component';

@Component({
  selector: 'app-order-list',
  standalone: true,  // ✅ No NgModule needed
  imports: [
    CommonModule,
    RouterModule,
    OrderCardComponent
  ],
  templateUrl: './order-list.component.html',
  styleUrls: ['./order-list.component.css']
})
export class OrderListComponent implements OnInit {
  orders$ = this.orderService.getOrders();

  constructor(private orderService: OrderService) {}

  ngOnInit() {
    // Component logic
  }
}
```

## Component Architecture Patterns

### Smart (Container) Components

Components that handle data fetching and state management:

```typescript
// features/order/pages/order-detail/order-detail.component.ts
@Component({
  selector: 'app-order-detail',
  standalone: true,
  imports: [CommonModule, OrderFormComponent],
  templateUrl: './order-detail.component.html'
})
export class OrderDetailComponent implements OnInit {
  orderId = input.required<string>();
  order$: Observable<Order>;
  loading$ = this.orderFacade.loading$;
  error$ = this.orderFacade.error$;

  constructor(private orderFacade: OrderFacadeService) {}

  ngOnInit() {
    this.order$ = this.orderFacade.getOrderById(this.orderId());
  }

  onSave(order: Order) {
    this.orderFacade.updateOrder(order);
  }
}
```

### Presentational (Dumb) Components

Reusable UI components with no dependencies on services:

```typescript
// features/order/components/order-card/order-card.component.ts
@Component({
  selector: 'app-order-card',
  standalone: true,
  imports: [CommonModule],
  templateUrl: './order-card.component.html'
})
export class OrderCardComponent {
  order = input.required<Order>();
  onEdit = output<Order>();
  onDelete = output<string>();

  edit() {
    this.onEdit.emit(this.order());
  }

  delete() {
    this.onDelete.emit(this.order().id);
  }
}
```

## Services & Dependency Injection

### Feature Service (Facade Pattern)

Central service for a feature:

```typescript
// features/order/services/order-facade.service.ts
@Injectable({ providedIn: 'root' })
export class OrderFacadeService {
  private readonly orderService = inject(OrderService);
  private readonly store = inject(OrderStore);

  orders$ = this.store.orders$;
  loading$ = this.store.loading$;
  error$ = this.store.error$;

  getOrderById(id: string): Observable<Order> {
    return this.orderService.getById(id).pipe(
      tap(order => this.store.setOrder(order)),
      catchError(error => {
        this.store.setError(error);
        return throwError(() => error);
      })
    );
  }

  updateOrder(order: Order): void {
    this.store.setLoading(true);
    this.orderService.update(order).subscribe({
      next: updated => this.store.setOrder(updated),
      error: err => this.store.setError(err)
    });
  }
}
```

### API Service

HTTP communication:

```typescript
// core/services/api.service.ts
@Injectable({ providedIn: 'root' })
export class ApiService {
  private readonly http = inject(HttpClient);
  private readonly config = inject(ApplicationConfig);

  get<T>(endpoint: string): Observable<T> {
    return this.http.get<T>(`${this.config.apiUrl}${endpoint}`);
  }

  post<T>(endpoint: string, body: any): Observable<T> {
    return this.http.post<T>(`${this.config.apiUrl}${endpoint}`, body);
  }

  put<T>(endpoint: string, body: any): Observable<T> {
    return this.http.put<T>(`${this.config.apiUrl}${endpoint}`, body);
  }

  delete<T>(endpoint: string): Observable<T> {
    return this.http.delete<T>(`${this.config.apiUrl}${endpoint}`);
  }
}
```

### Dependency Injection Configuration

Configure providers at appropriate level:

```typescript
// app.config.ts - Root level
import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { routes } from './app.routes';
import { authInterceptor } from './core/interceptors/auth.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(
      withInterceptors([authInterceptor])
    ),
    // Global services
  ]
};

// core/core.config.ts - Core services
export const CORE_PROVIDERS = [
  AuthService,
  ApiService,
  StorageService
];

// features/order/order.config.ts - Feature services
export const ORDER_FEATURE_PROVIDERS = [
  OrderService,
  OrderFacadeService,
  OrderStore
];
```

## State Management with Signals

Angular v17+ primitive for reactive state:

```typescript
// features/order/store/order.signal.ts
export class OrderStore {
  private readonly orderService = inject(OrderService);

  // State signals
  private readonly orders = signal<Order[]>([]);
  private readonly loading = signal(false);
  private readonly error = signal<string | null>(null);
  private readonly selectedOrder = signal<Order | null>(null);

  // Public read-only signals
  readonly orders$ = this.orders.asReadonly();
  readonly loading$ = this.loading.asReadonly();
  readonly error$ = this.error.asReadonly();
  readonly selectedOrder$ = this.selectedOrder.asReadonly();

  // Computed signal
  readonly orderCount = computed(() => this.orders().length);
  readonly hasOrders = computed(() => this.orderCount() > 0);

  // Actions
  loadOrders(): void {
    this.loading.set(true);
    this.orderService.getAll().subscribe({
      next: orders => {
        this.orders.set(orders);
        this.error.set(null);
      },
      error: err => this.error.set(err.message),
      finalize: () => this.loading.set(false)
    });
  }

  selectOrder(order: Order): void {
    this.selectedOrder.set(order);
  }

  addOrder(order: Order): void {
    this.orders.update(orders => [...orders, order]);
  }

  updateOrder(updated: Order): void {
    this.orders.update(orders =>
      orders.map(o => o.id === updated.id ? updated : o)
    );
  }

  clearError(): void {
    this.error.set(null);
  }
}
```

## Observable Patterns with RxJS

State management with Observables (alternative to Signals):

```typescript
// features/order/store/order.state.ts
@Injectable()
export class OrderState {
  private readonly orderService = inject(OrderService);
  private readonly reload$ = new Subject<void>();

  private readonly orders$ = this.reload$.pipe(
    startWith(undefined),
    switchMap(() => this.orderService.getAll()),
    shareReplay(1)
  );

  private readonly selectedOrder$ = new BehaviorSubject<Order | null>(null);
  private readonly loading$ = new BehaviorSubject(false);

  // Public observables
  readonly orders = this.orders$.asObservable();
  readonly selected = this.selectedOrder$.asObservable();
  readonly loading = this.loading$.asObservable();

  // Filtered orders
  readonly activeOrders$ = this.orders$.pipe(
    map(orders => orders.filter(o => o.status === 'active'))
  );

  selectOrder(order: Order): void {
    this.selectedOrder$.next(order);
  }

  reload(): void {
    this.reload$.next();
  }

  updateOrder(order: Order): void {
    this.loading$.next(true);
    this.orderService.update(order).subscribe({
      next: updated => {
        this.reload();
      },
      error: err => console.error(err),
      finalize: () => this.loading$.next(false)
    });
  }
}
```

## Routing Structure

Organized routing with feature modules:

```typescript
// app.routes.ts
export const routes: Routes = [
  {
    path: '',
    component: LayoutComponent,
    children: [
      {
        path: 'dashboard',
        loadComponent: () => import('./features/dashboard/pages/dashboard.component')
          .then(m => m.DashboardComponent)
      },
      {
        path: 'orders',
        loadChildren: () => import('./features/order/order.routes')
          .then(m => m.ORDER_ROUTES),
        canActivate: [authGuard]
      },
      {
        path: 'customers',
        loadChildren: () => import('./features/customer/customer.routes')
          .then(m => m.CUSTOMER_ROUTES),
        canActivate: [authGuard]
      }
    ]
  },
  {
    path: 'login',
    loadComponent: () => import('./features/auth/login.component')
      .then(m => m.LoginComponent)
  }
];

// features/order/order.routes.ts
export const ORDER_ROUTES: Routes = [
  {
    path: '',
    component: OrderListComponent
  },
  {
    path: ':id',
    component: OrderDetailComponent,
    canDeactivate: [canDeactivateGuard]
  },
  {
    path: 'new',
    component: OrderFormComponent
  }
];
```

## Interceptors & Guards

### HTTP Interceptor

```typescript
// core/interceptors/auth.interceptor.ts
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const token = authService.getToken();

  if (token) {
    req = req.clone({
      setHeaders: {
        Authorization: `Bearer ${token}`
      }
    });
  }

  return next(req).pipe(
    catchError(error => {
      if (error.status === 401) {
        authService.logout();
      }
      return throwError(() => error);
    })
  );
};
```

### Route Guard

```typescript
// core/guards/auth.guard.ts
export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) {
    return true;
  }

  router.navigate(['/login'], { queryParams: { returnUrl: state.url } });
  return false;
};

export const canDeactivateGuard: CanDeactivateFn<CanDeactivate> = 
  (component: CanDeactivate) => {
    return component.canDeactivate();
  };
```

## Testing Structure

Organize tests alongside components:

```
features/order/components/order-card/
├── order-card.component.ts
├── order-card.component.html
├── order-card.component.css
└── order-card.component.spec.ts

features/order/services/
├── order.service.ts
└── order.service.spec.ts
```

## Best Practices

- **Standalone by default:** Use standalone components (v14+)
- **Smart vs. Presentational:** Keep components focused - smart handle data, presentational handle UI
- **Services for logic:** Move business logic to services, keep components thin
- **Strong typing:** Use TypeScript interfaces for all data structures
- **Lazy loading:** Load feature modules on demand
- **OnPush detection:** Use `ChangeDetectionStrategy.OnPush` for performance
- **Unsubscribe:** Use `takeUntilDestroyed()` to clean up subscriptions
- **Reactive forms:** Use reactive forms over template-driven for complex forms
- **Environment config:** Use environment files for API URLs and configuration

## File Naming Conventions

```
Components:         *.component.ts      (order.component.ts)
Services:           *.service.ts        (order.service.ts)
Guards:             *.guard.ts          (auth.guard.ts)
Interceptors:       *.interceptor.ts    (error.interceptor.ts)
Pipes:              *.pipe.ts           (safe-html.pipe.ts)
Directives:         *.directive.ts      (highlight.directive.ts)
Models/Interfaces:  *.model.ts          (order.model.ts)
Routes:             *.routes.ts         (order.routes.ts)
Configuration:      *.config.ts         (app.config.ts)
Signals/State:      *.signal.ts         (order.signal.ts)
Tests:              *.spec.ts           (order.component.spec.ts)
```

## Related Concepts

- **RxJS:** Reactive Extensions for JavaScript
- **NgRx:** State management library (alternative to Signals)
- **Angular Material:** UI component library
- **Change Detection:** OnPush strategy for optimization
- **Dependency Injection:** Angular's IoC container
- **Lazy Loading:** Load modules/components on demand
- **Reactive Forms:** Form handling with better control

## References & Sources

### Official Angular Documentation
- Angular Official Guide: https://angular.io/guide
- Angular Style Guide: https://angular.io/guide/styleguide
- Angular Component Interaction: https://angular.io/guide/component-interaction
- Angular Dependency Injection: https://angular.io/guide/dependency-injection
- Angular Standalone Components: https://angular.io/guide/standalone-components
- Angular Signals: https://angular.io/guide/signals
- Angular Router: https://angular.io/guide/router

### Articles & Best Practices
- Angular Blog: https://blog.angular.io/
- Community Best Practices: https://angular.io/guide
- Smart and Presentational Components Pattern
- Feature-Based Application Architecture

### Related Technologies
- **RxJS:** https://rxjs.dev/
- **NgRx:** https://ngrx.io/
- **Angular Material:** https://material.angular.io/
- **TypeScript:** https://www.typescriptlang.org/

### Note
This guide reflects **modern Angular v20+ best practices** with standalone components. The structure is based on industry-standard patterns and the Angular team's recommendations, though Angular does not mandate a single rigid folder structure. Adjust based on your project's specific needs.
