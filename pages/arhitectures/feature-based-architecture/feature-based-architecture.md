# Feature-Based Architecture

Feature-Based Architecture organizes code around business features or functionalities rather than technical layers. Each feature is self-contained with its own components, services, routes, and styles. This approach makes it easy to add, remove, or maintain features independently.

## Key Concepts

- **Feature-Centric Organization:** Code grouped by business features
- **Self-Contained:** Each feature has everything it needs (components, services, routes)
- **Scalability:** Easy to add new features without affecting existing ones
- **Team Independence:** Teams can work on different features in parallel
- **Feature Autonomy:** Minimal dependencies between features
- **Co-location:** Related code is physically close in the file system

## Architecture Overview

```
src/features/
├── auth/                             # Authentication Feature
├── users/                            # User Management Feature
├── products/                         # Product Management Feature
├── orders/                           # Order Management Feature
├── payments/                         # Payment Processing Feature
└── notifications/                    # Notification System Feature
```

## Project Structure

### Angular/React Feature Module Structure

```
src/features/
├── auth/
│   ├── components/
│   │   ├── login/
│   │   │   ├── login.component.ts
│   │   │   ├── login.component.html
│   │   │   └── login.component.css
│   │   ├── signup/
│   │   │   └── signup.component.ts
│   │   └── password-reset/
│   │       └── password-reset.component.ts
│   ├── services/
│   │   ├── auth.service.ts
│   │   └── token.service.ts
│   ├── guards/
│   │   └── auth.guard.ts
│   ├── types/
│   │   └── auth.types.ts
│   ├── routes/
│   │   └── auth.routes.ts
│   └── auth.module.ts (or auth.config.ts for standalone)
├── users/
│   ├── components/
│   │   ├── user-list/
│   │   ├── user-detail/
│   │   └── user-profile/
│   ├── services/
│   │   └── user.service.ts
│   ├── types/
│   │   └── user.types.ts
│   └── routes/
│       └── user.routes.ts
└── orders/
    ├── components/
    │   ├── order-list/
    │   ├── order-detail/
    │   └── order-form/
    ├── services/
    │   └── order.service.ts
    ├── types/
    │   └── order.types.ts
    └── routes/
        └── order.routes.ts
```

### Express.js Feature Structure

```
src/features/
├── auth/
│   ├── auth.controller.ts
│   ├── auth.service.ts
│   ├── auth.routes.ts
│   ├── auth.middleware.ts
│   ├── auth.types.ts
│   └── auth.module.ts
├── users/
│   ├── users.controller.ts
│   ├── users.service.ts
│   ├── users.repository.ts
│   ├── users.routes.ts
│   ├── users.types.ts
│   └── users.module.ts
└── products/
    ├── products.controller.ts
    ├── products.service.ts
    ├── products.routes.ts
    ├── products.types.ts
    └── products.module.ts
```

## Feature Module Example

### Angular Feature Module (Standalone)

```typescript
// features/auth/auth.config.ts
import { importProvidersFrom } from '@angular/core';
import { HttpClientModule } from '@angular/common/http';
import { AuthService } from './services/auth.service';
import { AuthGuard } from './guards/auth.guard';

export const AUTH_PROVIDERS = [
  importProvidersFrom(HttpClientModule),
  AuthService,
  AuthGuard
];

// features/auth/routes/auth.routes.ts
import { Routes } from '@angular/router';
import { LoginComponent } from '../components/login/login.component';
import { SignupComponent } from '../components/signup/signup.component';

export const AUTH_ROUTES: Routes = [
  { path: 'login', component: LoginComponent },
  { path: 'signup', component: SignupComponent },
  { path: 'reset-password', component: PasswordResetComponent }
];

// features/auth/services/auth.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { BehaviorSubject } from 'rxjs';
import { User, LoginRequest } from '../types/auth.types';

@Injectable({ providedIn: 'root' })
export class AuthService {
  private currentUser = new BehaviorSubject<User | null>(null);
  public currentUser$ = this.currentUser.asObservable();

  constructor(private http: HttpClient) {}

  login(request: LoginRequest) {
    return this.http.post<User>('/api/auth/login', request);
  }

  logout() {
    this.currentUser.next(null);
  }
}

// app.routes.ts (root)
import { Routes } from '@angular/router';
import { AUTH_ROUTES } from './features/auth/routes/auth.routes';
import { AuthGuard } from './features/auth/guards/auth.guard';

export const routes: Routes = [
  { path: 'auth', children: AUTH_ROUTES },
  {
    path: 'users',
    loadChildren: () => import('./features/users/routes/user.routes')
      .then(m => m.USER_ROUTES),
    canActivate: [AuthGuard]
  },
  {
    path: 'orders',
    loadChildren: () => import('./features/orders/routes/order.routes')
      .then(m => m.ORDER_ROUTES),
    canActivate: [AuthGuard]
  }
];
```

### Express.js Feature Module

```typescript
// features/auth/auth.module.ts
import { Router } from 'express';
import { AuthController } from './auth.controller';
import { AuthService } from './auth.service';
import { AuthMiddleware } from './auth.middleware';

export function createAuthModule(): Router {
  const router = Router();
  const authService = new AuthService();
  const authController = new AuthController(authService);
  const authMiddleware = new AuthMiddleware(authService);

  router.post('/login', (req, res) => authController.login(req, res));
  router.post('/logout', (req, res) => authController.logout(req, res));
  router.post('/signup', (req, res) => authController.signup(req, res));

  return router;
}

// app.ts
import { createAuthModule } from './features/auth/auth.module';
import { createUsersModule } from './features/users/users.module';
import { createOrdersModule } from './features/orders/orders.module';

const app = express();

app.use('/api/auth', createAuthModule());
app.use('/api/users', createUsersModule());
app.use('/api/orders', createOrdersModule());
```

## Inter-Feature Communication

### Method 1: Shared Services

Services can be injected across features:

```typescript
// features/auth/services/auth.service.ts
@Injectable({ providedIn: 'root' })
export class AuthService {
  getCurrentUser(): User | null { }
}

// features/users/services/user.service.ts
@Injectable({ providedIn: 'root' })
export class UserService {
  constructor(private authService: AuthService) {}

  getProfile() {
    const user = this.authService.getCurrentUser();
    // ...
  }
}
```

### Method 2: Event Bus / Pub-Sub

Loose coupling using events:

```typescript
// shared/event-bus.ts
import { Subject } from 'rxjs';

export class EventBus {
  private events$ = new Subject<AppEvent>();

  emit(event: AppEvent) {
    this.events$.next(event);
  }

  subscribe(handler: (event: AppEvent) => void) {
    return this.events$.subscribe(handler);
  }
}

// features/auth/services/auth.service.ts
@Injectable()
export class AuthService {
  constructor(private eventBus: EventBus) {}

  login(credentials: LoginRequest) {
    // ... login logic ...
    this.eventBus.emit(new UserLoggedInEvent(user));
  }
}

// features/notifications/services/notification.service.ts
@Injectable()
export class NotificationService {
  constructor(private eventBus: EventBus) {
    this.eventBus.subscribe((event) => {
      if (event instanceof UserLoggedInEvent) {
        this.sendWelcomeNotification(event.user);
      }
    });
  }
}
```

### Method 3: Facades / APIs

Each feature exposes a public API:

```typescript
// features/auth/auth.facade.ts
import { Injectable } from '@angular/core';
import { AuthService } from './services/auth.service';
import { User } from './types/auth.types';

@Injectable({ providedIn: 'root' })
export class AuthFacade {
  constructor(private authService: AuthService) {}

  login(email: string, password: string) {
    return this.authService.login(email, password);
  }

  logout() {
    return this.authService.logout();
  }

  getCurrentUser() {
    return this.authService.getCurrentUser();
  }
}

// features/users/components/user-detail.component.ts
export class UserDetailComponent {
  constructor(private authFacade: AuthFacade) {
    this.currentUser = this.authFacade.getCurrentUser();
  }
}
```

## Feature Rules

```typescript
// CLAUDE.md - Feature-Based Architecture Rules

## Architecture: Feature-Based

### Rules:
1. Each feature is self-contained in `src/features/{featureName}/`
2. Feature includes: components, services, routes, types, guards
3. Features communicate via:
   - Shared services (AuthService, ConfigService)
   - Event Bus for loosely coupled events
   - Facade pattern for public API
4. Never import from another feature's internal files:
   - ❌ `import from '../users/services/user.service'`
   - ✅ `import from '../users/user.facade'` or event bus
5. Maximum file size: 400 lines
6. Each feature must have:
   - index.ts (barrel export)
   - types.ts (interfaces)
   - routes.ts or module.ts
   - services and components

### Dependencies:
- Features depend on shared/ (shared services, utilities)
- Features DO NOT depend on other features directly
- Inter-feature communication via EventBus or Facades only
```

## Feature Scaling

### Small Features (< 10 files)

```
features/auth/
├── components/
├── services/
├── types/
├── auth.module.ts
└── index.ts
```

### Large Features (> 20 files)

```
features/orders/
├── components/
│   ├── list/
│   ├── detail/
│   └── form/
├── services/
├── repositories/
├── types/
├── routes/
├── guards/
├── pipes/
├── store/ (NgRx or state management)
├── orders.module.ts
└── index.ts
```

## Testing Structure

```typescript
features/auth/
├── services/
│   ├── auth.service.ts
│   └── auth.service.spec.ts
├── guards/
│   ├── auth.guard.ts
│   └── auth.guard.spec.ts
└── components/
    ├── login/
    │   ├── login.component.ts
    │   └── login.component.spec.ts
```

## Advantages

- **Easy Navigation:** All feature code is together
- **Scalability:** Add new features without modifying existing ones
- **Team Parallelization:** Teams can work on different features
- **Feature Removal:** Delete entire feature folder if needed
- **Maintainability:** Feature contains all its dependencies
- **Onboarding:** New developers understand features quickly
- **Lazy Loading:** Features can be lazy-loaded easily

## Disadvantages

- **Code Duplication:** Similar logic may exist in multiple features
- **Cross-Feature Logic:** Harder to refactor code used by multiple features
- **Shared State:** Multiple features may need same data
- **Learning Curve:** Need clear boundaries and communication patterns
- **Folder Proliferation:** Can create many folders at root level

## When to Use

- Medium to large applications with many features
- Team working on different features independently
- Features are added/removed frequently
- Each feature is relatively independent
- Long-term project with evolving requirements

## Best Practices

- **Define Clear Boundaries:** Explicit feature interfaces
- **Minimize Dependencies:** Features should be loosely coupled
- **Shared Services:** Common services in shared/ folder
- **Event Bus:** Use for feature-to-feature communication
- **Documentation:** Document feature public API
- **Consistent Structure:** All features follow same pattern
- **Feature Facades:** Public interface for each feature
- **Testing:** Test features independently

## Comparison: Feature-Based vs. Layered

| Aspect | Feature-Based | Layered (Clean Arch) |
|--------|---------------|---------------------|
| Organization | By business feature | By technical layer |
| Navigation | Quick - all files together | Jumping between folders |
| Scaling | Easy - add new features | Complex - affects many layers |
| Testing | Feature in isolation | Mocking many dependencies |
| Team Work | One team per feature | Shared layers |
| File Location | Predictable (feature folder) | Spread across 4+ layers |
| Code Reuse | Harder - may duplicate | Easier - shared layer |

## Related Patterns

- **Monorepo:** Organize multiple features as packages
- **[Facade Pattern](../../patterns/facade-pattern/facade.md):** Public API for each feature
- **Event Bus:** Loose coupling between features
- **Module Pattern:** Encapsulation within features
- **[Separation of Concerns](../separation-of-concerns/separation-of-concerns.md):** Feature-based is a vertical application of this same underlying principle

## References & Sources

- Angular Feature Modules: https://angular.io/guide/feature-modules
- Feature-Based Project Structure: https://github.com/angular/angular-cli/wiki
- React Feature Folder Organization: https://react.dev/learn
- NestJS Module Pattern: https://docs.nestjs.com/modules
- Domain-Driven Design - bounded contexts parallel to features
