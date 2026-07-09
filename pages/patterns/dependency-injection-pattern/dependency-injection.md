# Dependency Injection (DI) Pattern

Dependency Injection is a design pattern that deals with how components get hold of their dependencies. The pattern states that objects or functions should receive other objects or functions that they depend on, rather than creating them internally. This loose coupling improves testability, maintainability, and flexibility.

## Problem

- Classes create their own dependencies (tight coupling)
- Hard to test because dependencies are hardcoded
- Difficult to swap implementations
- Changes to dependencies require changing the class
- Difficult to manage complex dependency graphs

## Solution

Provide dependencies from the outside (inject them) rather than having objects create them internally.

## Three Types of Dependency Injection

### 1. Constructor Injection

Dependencies are provided through the constructor:

```typescript
// Without DI - Tight Coupling
class UserService {
  private database: Database;

  constructor() {
    this.database = new PostgresDatabase(); // Hardcoded dependency
  }

  getUser(id: string): User {
    return this.database.query(`SELECT * FROM users WHERE id = ${id}`);
  }
}

// With Constructor DI - Loose Coupling
class UserService {
  constructor(private database: Database) {} // Injected dependency

  getUser(id: string): User {
    return this.database.query(`SELECT * FROM users WHERE id = ${id}`);
  }
}

// Usage
const database = new PostgresDatabase();
const userService = new UserService(database);
```

### 2. Setter Injection

Dependencies are provided through setter methods:

```typescript
interface Logger {
  log(message: string): void;
}

class Application {
  private logger: Logger | null = null;

  setLogger(logger: Logger): void {
    this.logger = logger;
  }

  run(): void {
    this.logger?.log('Application started');
  }
}

// Usage
const app = new Application();
app.setLogger(new ConsoleLogger());
app.run();
```

### 3. Interface/Parameter Injection

Dependencies are passed as function parameters:

```typescript
interface Database {
  query(sql: string): Promise<any>;
}

function getUserById(id: string, database: Database): Promise<User> {
  return database.query(`SELECT * FROM users WHERE id = ${id}`);
}

// Usage
const database = new PostgresDatabase();
const user = await getUserById('123', database);
```

## Real-World Example: Service Dependency Injection

```typescript
// Interfaces (Contracts)
interface IEmailService {
  sendEmail(to: string, subject: string, body: string): Promise<void>;
}

interface ILogger {
  log(message: string): void;
}

interface IUserRepository {
  findById(id: string): Promise<User | null>;
  save(user: User): Promise<void>;
}

// Implementations
class EmailService implements IEmailService {
  async sendEmail(to: string, subject: string, body: string): Promise<void> {
    console.log(`Sending email to ${to}: ${subject}`);
    // Send email
  }
}

class ConsoleLogger implements ILogger {
  log(message: string): void {
    console.log(`[LOG] ${message}`);
  }
}

class PostgresUserRepository implements IUserRepository {
  async findById(id: string): Promise<User | null> {
    // Query database
    return null;
  }

  async save(user: User): Promise<void> {
    // Save to database
  }
}

// Service with DI
class UserService {
  constructor(
    private userRepository: IUserRepository,
    private emailService: IEmailService,
    private logger: ILogger
  ) {}

  async registerUser(email: string, name: string): Promise<User> {
    this.logger.log(`Registering user: ${email}`);

    const user: User = { id: '1', email, name };
    await this.userRepository.save(user);

    this.logger.log(`Sending welcome email to ${email}`);
    await this.emailService.sendEmail(
      email,
      'Welcome!',
      `Hello ${name}, welcome to our service!`
    );

    return user;
  }
}

// Usage with Dependency Injection
const emailService = new EmailService();
const logger = new ConsoleLogger();
const userRepository = new PostgresUserRepository();

const userService = new UserService(userRepository, emailService, logger);
await userService.registerUser('john@example.com', 'John Doe');
```

## Dependency Injection Container

Automatically manage and inject dependencies:

```typescript
class ServiceContainer {
  private services: Map<string, () => any> = new Map();

  register(name: string, factory: () => any): void {
    this.services.set(name, factory);
  }

  get<T>(name: string): T {
    const factory = this.services.get(name);
    if (!factory) {
      throw new Error(`Service ${name} not registered`);
    }
    return factory() as T;
  }

  getSingleton<T>(name: string): T {
    let instance: T | null = null;
    return (() => {
      if (!instance) {
        instance = this.get<T>(name);
      }
      return instance;
    })() as T;
  }
}

// Usage
const container = new ServiceContainer();

container.register('emailService', () => new EmailService());
container.register('logger', () => new ConsoleLogger());
container.register('userRepository', () => new PostgresUserRepository());

container.register('userService', () => {
  return new UserService(
    container.get('userRepository'),
    container.get('emailService'),
    container.get('logger')
  );
});

const userService = container.get<UserService>('userService');
```

## Framework-Based DI (NestJS)

```typescript
import { Injectable } from '@nestjs/common';

@Injectable()
export class UserService {
  constructor(
    private userRepository: UserRepository,
    private emailService: EmailService,
    private logger: LoggerService
  ) {}

  async registerUser(email: string, name: string): Promise<User> {
    this.logger.log(`Registering user: ${email}`);
    // Implementation
  }
}

// NestJS automatically injects dependencies
@Controller('users')
export class UserController {
  constructor(private userService: UserService) {}

  @Post('register')
  async register(@Body() dto: RegisterDto) {
    return this.userService.registerUser(dto.email, dto.name);
  }
}
```

## Testing with Dependency Injection

```typescript
// Mock implementations for testing
class MockEmailService implements IEmailService {
  async sendEmail(): Promise<void> {
    // Mock implementation - do nothing
  }
}

class MockLogger implements ILogger {
  log(message: string): void {
    // Mock implementation - do nothing
  }
}

class MockUserRepository implements IUserRepository {
  async findById(id: string): Promise<User | null> {
    return { id, email: 'test@example.com', name: 'Test User' };
  }

  async save(user: User): Promise<void> {
    // Mock implementation
  }
}

// Unit test
describe('UserService', () => {
  let userService: UserService;
  let mockEmailService: MockEmailService;
  let mockLogger: MockLogger;
  let mockUserRepository: MockUserRepository;

  beforeEach(() => {
    mockEmailService = new MockEmailService();
    mockLogger = new MockLogger();
    mockUserRepository = new MockUserRepository();

    userService = new UserService(
      mockUserRepository,
      mockEmailService,
      mockLogger
    );
  });

  test('should register a user', async () => {
    const user = await userService.registerUser('john@example.com', 'John');
    expect(user.email).toBe('john@example.com');
    expect(user.name).toBe('John');
  });
});
```

## Advantages

- **Loose Coupling:** Classes don't depend on concrete implementations
- **Testability:** Easy to inject mock implementations for testing
- **Flexibility:** Swap implementations without changing code
- **Reusability:** Same service with different dependencies
- **Maintainability:** Changes isolated to factory or container
- **Separation of Concerns:** Object creation separate from usage

## Disadvantages

- **Complexity:** Adds abstraction and indirection
- **Learning Curve:** Developers need to understand DI concepts
- **Boilerplate:** More interfaces and configurations
- **Performance:** Extra function calls for dependency resolution
- **Debugging:** Harder to trace dependency chain

## Best Practices

- **Depend on Abstractions:** Inject interfaces, not concrete classes
- **Constructor Injection:** Preferred method for required dependencies
- **Avoid Service Locator:** Anti-pattern that hides dependencies
- **Lazy Loading:** Load expensive dependencies only when needed
- **Dependency Graph:** Keep dependency graph simple and acyclic
- **Use Containers:** For complex applications with many dependencies
- **Type Safety:** Use TypeScript for compile-time dependency checking

## Related Patterns

- **Service Locator:** Anti-pattern, don't use
- **Factory Pattern:** Used with DI containers
- **Singleton Pattern:** Manage singleton instances with DI
- **Strategy Pattern:** Inject different strategies
- **Decorator Pattern:** Wrap injected dependencies

## References & Sources

- Martin Fowler - Inversion of Control: https://martinfowler.com/bliki/InversionOfControl.html
- Martin Fowler - Service Locator: https://martinfowler.com/articles/injection.html
- NestJS Documentation - Providers: https://docs.nestjs.com/providers
- Spring Framework - Dependency Injection: https://spring.io/
- Angular - Dependency Injection: https://angular.io/guide/dependency-injection
