# ElysiaJS Architecture

ElysiaJS architecture emphasizes modularity, type safety, and clean separation of concerns. This guide covers organizing large-scale backend applications built with ElysiaJS and Bun.

## Key Principles

- **Module-Based Organization:** Group related features together
- **Type-Safe Routes:** End-to-end type inference from request to response
- **Dependency Injection:** Share services across modules
- **Middleware Composition:** Stack middleware for cross-cutting concerns
- **Plugin Architecture:** Encapsulate functionality in reusable plugins
- **Service Layer:** Separate business logic from route handlers

## Project Structure (Feature-Based)

Organize by features/domains with clear responsibility separation:

```
src/
├── modules/                          # Feature modules
│   ├── auth/                         # Authentication Feature
│   │   ├── auth.controller.ts        # Route handlers
│   │   ├── auth.service.ts           # Business logic
│   │   ├── auth.guard.ts             # Authentication middleware
│   │   ├── auth.types.ts             # Types and DTOs
│   │   ├── auth.schema.ts            # Validation schemas
│   │   └── auth.module.ts            # Module export
│   ├── users/                        # User Management Feature
│   │   ├── users.controller.ts
│   │   ├── users.service.ts
│   │   ├── users.types.ts
│   │   ├── users.schema.ts
│   │   └── users.module.ts
│   ├── posts/                        # Posts Feature
│   │   ├── posts.controller.ts
│   │   ├── posts.service.ts
│   │   ├── posts.repository.ts       # Data access layer
│   │   ├── posts.types.ts
│   │   └── posts.module.ts
│   └── notifications/
│       └── ...
├── middleware/                       # Global middleware
│   ├── logging.middleware.ts
│   ├── error-handler.middleware.ts
│   ├── cors.middleware.ts
│   └── auth.middleware.ts
├── guards/                           # Request guards
│   ├── auth.guard.ts
│   ├── role.guard.ts
│   └── validation.guard.ts
├── services/                         # Shared services
│   ├── database.service.ts           # Database client
│   ├── cache.service.ts              # Caching layer
│   ├── logger.service.ts             # Logging
│   ├── email.service.ts              # External services
│   └── storage.service.ts            # File storage
├── types/                            # Global types
│   ├── index.ts
│   ├── api.types.ts
│   ├── dto.types.ts
│   └── errors.types.ts
├── schemas/                          # Shared validation schemas
│   ├── common.schema.ts
│   ├── pagination.schema.ts
│   └── response.schema.ts
├── utils/                            # Utility functions
│   ├── validators.ts
│   ├── formatters.ts
│   ├── helpers.ts
│   └── constants.ts
├── config/                           # Configuration
│   ├── database.config.ts
│   ├── cache.config.ts
│   ├── app.config.ts
│   └── env.ts                        # Environment variables
├── app.ts                            # App initialization
└── server.ts                         # Entry point
```

## Module Structure

Each feature module follows a consistent pattern:

### Controller (Route Handlers)

```typescript
// modules/users/users.controller.ts
import Elysia, { t } from 'elysia';
import { UsersService } from './users.service';

export class UsersController {
  private usersService: UsersService;

  constructor(usersService: UsersService) {
    this.usersService = usersService;
  }

  setupRoutes(app: Elysia) {
    return app
      .get('/users', () => this.usersService.getAllUsers())
      .get('/users/:id', ({ params: { id } }) => 
        this.usersService.getUserById(Number(id))
      )
      .post('/users', ({ body }) => 
        this.usersService.createUser(body)
      , {
        body: t.Object({
          name: t.String(),
          email: t.String({ format: 'email' })
        })
      })
      .put('/users/:id', ({ params: { id }, body }) =>
        this.usersService.updateUser(Number(id), body)
      )
      .delete('/users/:id', ({ params: { id } }) =>
        this.usersService.deleteUser(Number(id))
      );
  }
}
```

### Service (Business Logic)

```typescript
// modules/users/users.service.ts
import { UsersRepository } from './users.repository';
import { User, CreateUserDTO, UpdateUserDTO } from './users.types';

export class UsersService {
  constructor(private usersRepository: UsersRepository) {}

  async getAllUsers(): Promise<User[]> {
    return this.usersRepository.findAll();
  }

  async getUserById(id: number): Promise<User | null> {
    return this.usersRepository.findById(id);
  }

  async createUser(dto: CreateUserDTO): Promise<User> {
    // Business logic: validation, transformation, etc.
    if (await this.usersRepository.findByEmail(dto.email)) {
      throw new Error('Email already exists');
    }
    return this.usersRepository.create(dto);
  }

  async updateUser(id: number, dto: UpdateUserDTO): Promise<User> {
    const user = await this.usersRepository.findById(id);
    if (!user) throw new Error('User not found');
    return this.usersRepository.update(id, dto);
  }

  async deleteUser(id: number): Promise<boolean> {
    return this.usersRepository.delete(id);
  }
}
```

### Repository (Data Access)

```typescript
// modules/users/users.repository.ts
import { PrismaClient } from '@prisma/client';
import { User, CreateUserDTO, UpdateUserDTO } from './users.types';

export class UsersRepository {
  constructor(private prisma: PrismaClient) {}

  async findAll(): Promise<User[]> {
    return this.prisma.user.findMany();
  }

  async findById(id: number): Promise<User | null> {
    return this.prisma.user.findUnique({ where: { id } });
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.prisma.user.findUnique({ where: { email } });
  }

  async create(data: CreateUserDTO): Promise<User> {
    return this.prisma.user.create({ data });
  }

  async update(id: number, data: UpdateUserDTO): Promise<User> {
    return this.prisma.user.update({
      where: { id },
      data
    });
  }

  async delete(id: number): Promise<boolean> {
    await this.prisma.user.delete({ where: { id } });
    return true;
  }
}
```

### Types & DTOs

```typescript
// modules/users/users.types.ts
export interface User {
  id: number;
  name: string;
  email: string;
  createdAt: Date;
  updatedAt: Date;
}

export interface CreateUserDTO {
  name: string;
  email: string;
}

export interface UpdateUserDTO {
  name?: string;
  email?: string;
}

export interface UserResponse {
  id: number;
  name: string;
  email: string;
}
```

### Schemas

```typescript
// modules/users/users.schema.ts
import { t } from 'elysia';

export const CreateUserSchema = t.Object({
  name: t.String({ minLength: 1, maxLength: 255 }),
  email: t.String({ format: 'email' })
});

export const UpdateUserSchema = t.Partial(CreateUserSchema);

export const UserResponseSchema = t.Object({
  id: t.Number(),
  name: t.String(),
  email: t.String(),
  createdAt: t.Date(),
  updatedAt: t.Date()
});
```

### Module Export

```typescript
// modules/users/users.module.ts
import Elysia from 'elysia';
import { PrismaClient } from '@prisma/client';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';
import { UsersRepository } from './users.repository';

export const usersModule = (prisma: PrismaClient) => {
  const repository = new UsersRepository(prisma);
  const service = new UsersService(repository);
  const controller = new UsersController(service);

  return new Elysia({ prefix: '/api' })
    .derive(() => ({ usersService: service }))
    .get('/users', () => controller.setupRoutes(new Elysia()).routes);
};

// Better approach: Plugin-based module
export const usersModulePlugin = (app: Elysia, prisma: PrismaClient) => {
  const repository = new UsersRepository(prisma);
  const service = new UsersService(repository);
  const controller = new UsersController(service);

  return controller.setupRoutes(app);
};
```

## Dependency Injection

### Service Container Pattern

```typescript
// services/container.ts
import { PrismaClient } from '@prisma/client';
import { UsersService } from '../modules/users/users.service';
import { PostsService } from '../modules/posts/posts.service';
import { AuthService } from '../modules/auth/auth.service';

export class ServiceContainer {
  private prisma: PrismaClient;
  private usersService: UsersService;
  private postsService: PostsService;
  private authService: AuthService;

  constructor() {
    this.prisma = new PrismaClient();
    
    // Initialize services with dependencies
    this.usersService = new UsersService(
      new UsersRepository(this.prisma)
    );
    this.postsService = new PostsService(
      new PostsRepository(this.prisma),
      this.usersService
    );
    this.authService = new AuthService(this.usersService);
  }

  getUsersService(): UsersService {
    return this.usersService;
  }

  getPostsService(): PostsService {
    return this.postsService;
  }

  getAuthService(): AuthService {
    return this.authService;
  }

  async disconnect(): Promise<void> {
    await this.prisma.$disconnect();
  }
}
```

### Using in App

```typescript
// app.ts
import Elysia from 'elysia';
import { ServiceContainer } from './services/container';
import { usersModulePlugin } from './modules/users/users.module';
import { postsModulePlugin } from './modules/posts/posts.module';

const container = new ServiceContainer();

export const createApp = () => {
  return new Elysia()
    .state('container', container)
    .use((app) => usersModulePlugin(app, container.getUsersService()))
    .use((app) => postsModulePlugin(app, container.getPostsService()))
    .onStop(() => container.disconnect());
};
```

## Plugin Architecture

### Creating Reusable Plugins

```typescript
// plugins/auth.plugin.ts
import Elysia, { t } from 'elysia';
import { jwt } from '@elysiajs/jwt';
import { AuthService } from '../modules/auth/auth.service';

export const createAuthPlugin = (authService: AuthService) => {
  return (app: Elysia) => {
    return app
      .use(jwt({
        name: 'jwt',
        secret: process.env.JWT_SECRET!
      }))
      .post('/auth/login', async ({ body, jwt: jwtService }) => {
        const user = await authService.authenticate(body.email, body.password);
        if (!user) return { error: 'Invalid credentials' };
        
        const token = await jwtService.sign({ userId: user.id, role: user.role });
        return { token, user };
      }, {
        body: t.Object({
          email: t.String({ format: 'email' }),
          password: t.String()
        })
      })
      .post('/auth/logout', async ({ jwt: jwtService, headers }) => {
        // Logout logic
        return { message: 'Logged out' };
      });
  };
};

// plugins/database.plugin.ts
import Elysia from 'elysia';
import { PrismaClient } from '@prisma/client';

export const createDatabasePlugin = () => {
  const prisma = new PrismaClient();

  return (app: Elysia) => {
    return app
      .state('db', prisma)
      .onStop(() => prisma.$disconnect());
  };
};

// app.ts
const app = new Elysia()
  .use(createDatabasePlugin())
  .use(createAuthPlugin(authService))
  .listen(3000);
```

## Middleware & Guards

### Authentication Middleware

```typescript
// middleware/auth.middleware.ts
import Elysia from 'elysia';

export const createAuthMiddleware = (app: Elysia) => {
  return app.guard({
    headers: { authorization: 'string' }
  }, (app) =>
    app.derive(async ({ headers, jwt: jwtService }) => {
      const token = headers.authorization.split(' ')[1];
      try {
        const user = await jwtService.verify(token);
        return { user };
      } catch (error) {
        throw new Error('Unauthorized');
      }
    })
  );
};
```

### Logging Middleware

```typescript
// middleware/logging.middleware.ts
import Elysia from 'elysia';

export const createLoggingMiddleware = (app: Elysia) => {
  return app.derive(({ request, store }) => {
    const startTime = Date.now();
    const logger = store.logger;

    logger.info(`[${request.method}] ${request.url}`);

    return {
      startTime,
      requestId: crypto.randomUUID()
    };
  });
};
```

## Error Handling

### Centralized Error Handler

```typescript
// middleware/error-handler.middleware.ts
import Elysia from 'elysia';

export class AppError extends Error {
  constructor(
    public statusCode: number,
    message: string,
    public code?: string
  ) {
    super(message);
  }
}

export const createErrorHandler = (app: Elysia) => {
  return app.onError(({ error, code, set }) => {
    if (error instanceof AppError) {
      set.status = error.statusCode;
      return {
        error: {
          message: error.message,
          code: error.code,
          statusCode: error.statusCode
        }
      };
    }

    if (code === 'VALIDATION') {
      set.status = 400;
      return {
        error: {
          message: 'Validation failed',
          code: 'VALIDATION_ERROR',
          statusCode: 400
        }
      };
    }

    set.status = 500;
    return {
      error: {
        message: 'Internal server error',
        code: 'INTERNAL_ERROR',
        statusCode: 500
      }
    };
  });
};
```

## Best Practices

- **Single Responsibility:** Each file/class has one purpose
- **Dependency Injection:** Pass dependencies, don't create them
- **Service Layer:** Separate business logic from HTTP layer
- **Type Safety:** Leverage TypeScript for compile-time safety
- **Validation:** Use schemas at HTTP boundaries
- **Error Handling:** Implement comprehensive error handling
- **Logging:** Log important events and errors
- **Environment Variables:** Use .env for configuration
- **Testing:** Test services independently from routes
- **Module Organization:** Keep features self-contained and portable

## File Naming Conventions

```
Controllers:        *.controller.ts      (users.controller.ts)
Services:           *.service.ts         (users.service.ts)
Repositories:       *.repository.ts      (users.repository.ts)
Types/DTOs:         *.types.ts           (users.types.ts)
Schemas:            *.schema.ts          (users.schema.ts)
Modules:            *.module.ts          (users.module.ts)
Middleware:         *.middleware.ts      (auth.middleware.ts)
Guards:             *.guard.ts           (auth.guard.ts)
Plugins:            *.plugin.ts          (auth.plugin.ts)
Tests:              *.test.ts            (users.service.test.ts)
Config:             *.config.ts          (app.config.ts)
```

## References & Sources

### Official Documentation
- ElysiaJS: https://elysiajs.com/
- Bun: https://bun.sh/
- Prisma: https://www.prisma.io/

### Articles & Guides
- Clean Architecture Principles
- Domain-Driven Design for Backend Services
- Dependency Injection Patterns
- Repository Pattern
- Service Layer Architecture

### Related Technologies
- **Prisma:** Database ORM
- **Zod:** Schema validation (alternative to Elysia.t)
- **TypeScript:** Type-safe development

### Note
This architecture emphasizes modularity, type safety, and separation of concerns. Adapt based on your application's scale and requirements.
