# Repository Pattern

The Repository Pattern is a structural pattern that provides an abstraction layer between the domain/business logic and data mapping layers. It acts as an in-memory collection of domain objects, isolating business logic from data access details.

## Problem

- Business logic tightly coupled to database implementation
- Hard to test business logic without real database
- Difficult to switch databases
- Data access logic scattered throughout the application
- No clear separation between domain and infrastructure

## Solution

Create a repository interface that abstracts data access, allowing the domain layer to work with domain objects without knowing about database details.

## Structure

```
┌──────────────────┐
│ Domain Layer     │
│ (Business Logic) │
└────────┬─────────┘
         │ uses
┌────────▼──────────────────┐
│ Repository Interface      │
│ (Contract/Abstraction)    │
└────────┬─────────────────┘
         │ implemented by
┌────────▼──────────────────┐
│ Concrete Repository       │
│ (Data Access Logic)       │
└────────┬─────────────────┘
         │ uses
┌────────▼──────────────────┐
│ Infrastructure Layer      │
│ (Database, ORM)           │
└───────────────────────────┘
```

## Generic Repository Pattern

```typescript
// Domain Entity
interface Entity {
  id: string;
}

// Repository Interface
interface IRepository<T extends Entity> {
  findById(id: string): Promise<T | null>;
  findAll(): Promise<T[]>;
  save(entity: T): Promise<T>;
  update(id: string, entity: Partial<T>): Promise<T>;
  delete(id: string): Promise<boolean>;
}

// Base Repository Implementation
abstract class BaseRepository<T extends Entity> implements IRepository<T> {
  protected data: Map<string, T> = new Map();

  async findById(id: string): Promise<T | null> {
    return this.data.get(id) || null;
  }

  async findAll(): Promise<T[]> {
    return Array.from(this.data.values());
  }

  async save(entity: T): Promise<T> {
    this.data.set(entity.id, entity);
    return entity;
  }

  async update(id: string, updates: Partial<T>): Promise<T> {
    const entity = await this.findById(id);
    if (!entity) throw new Error(`Entity with id ${id} not found`);
    const updated = { ...entity, ...updates };
    this.data.set(id, updated);
    return updated;
  }

  async delete(id: string): Promise<boolean> {
    return this.data.delete(id);
  }
}
```

## Domain Entity with Repository

```typescript
// Domain Entity
interface User extends Entity {
  name: string;
  email: string;
  createdAt: Date;
}

// Repository Interface
interface IUserRepository extends IRepository<User> {
  findByEmail(email: string): Promise<User | null>;
  findByRole(role: string): Promise<User[]>;
}

// Concrete Repository
class UserRepository extends BaseRepository<User> implements IUserRepository {
  async findByEmail(email: string): Promise<User | null> {
    for (const user of this.data.values()) {
      if (user.email === email) {
        return user;
      }
    }
    return null;
  }

  async findByRole(role: string): Promise<User[]> {
    // Implementation would depend on actual data structure
    return [];
  }
}

// Domain Service using Repository
class UserService {
  constructor(private userRepository: IUserRepository) {}

  async createUser(name: string, email: string): Promise<User> {
    // Business logic
    const existing = await this.userRepository.findByEmail(email);
    if (existing) {
      throw new Error('Email already registered');
    }

    const user: User = {
      id: Math.random().toString(),
      name,
      email,
      createdAt: new Date()
    };

    return this.userRepository.save(user);
  }

  async getUser(id: string): Promise<User | null> {
    return this.userRepository.findById(id);
  }

  async updateUserEmail(id: string, newEmail: string): Promise<User> {
    const existing = await this.userRepository.findByEmail(newEmail);
    if (existing && existing.id !== id) {
      throw new Error('Email already in use');
    }

    return this.userRepository.update(id, { email: newEmail });
  }
}

// Usage
const userRepository = new UserRepository();
const userService = new UserService(userRepository);

userService.createUser('John Doe', 'john@example.com').then(user => {
  console.log('User created:', user);
});
```

## Database Repository with ORM

```typescript
// Using Prisma
import { PrismaClient, User as PrismaUser } from '@prisma/client';

interface User extends Entity {
  name: string;
  email: string;
  role: string;
  createdAt: Date;
}

class UserRepositoryPrisma implements IUserRepository {
  constructor(private prisma: PrismaClient) {}

  async findById(id: string): Promise<User | null> {
    const user = await this.prisma.user.findUnique({
      where: { id }
    });
    return user ? this.mapPrismaToUser(user) : null;
  }

  async findAll(): Promise<User[]> {
    const users = await this.prisma.user.findMany();
    return users.map(u => this.mapPrismaToUser(u));
  }

  async findByEmail(email: string): Promise<User | null> {
    const user = await this.prisma.user.findUnique({
      where: { email }
    });
    return user ? this.mapPrismaToUser(user) : null;
  }

  async findByRole(role: string): Promise<User[]> {
    const users = await this.prisma.user.findMany({
      where: { role }
    });
    return users.map(u => this.mapPrismaToUser(u));
  }

  async save(entity: User): Promise<User> {
    const created = await this.prisma.user.create({
      data: {
        id: entity.id,
        name: entity.name,
        email: entity.email,
        role: entity.role
      }
    });
    return this.mapPrismaToUser(created);
  }

  async update(id: string, updates: Partial<User>): Promise<User> {
    const updated = await this.prisma.user.update({
      where: { id },
      data: updates
    });
    return this.mapPrismaToUser(updated);
  }

  async delete(id: string): Promise<boolean> {
    await this.prisma.user.delete({
      where: { id }
    });
    return true;
  }

  private mapPrismaToUser(prismaUser: PrismaUser): User {
    return {
      id: prismaUser.id,
      name: prismaUser.name,
      email: prismaUser.email,
      role: prismaUser.role,
      createdAt: prismaUser.createdAt
    };
  }
}
```

## Specification Pattern with Repository

```typescript
// Specification Interface
interface Specification<T> {
  isSatisfiedBy(candidate: T): boolean;
}

// Concrete Specifications
class EmailSpecification implements Specification<User> {
  constructor(private email: string) {}

  isSatisfiedBy(user: User): boolean {
    return user.email === this.email;
  }
}

class RoleSpecification implements Specification<User> {
  constructor(private role: string) {}

  isSatisfiedBy(user: User): boolean {
    return user.role === this.role;
  }
}

class CompositeSpecification<T> implements Specification<T> {
  constructor(
    private left: Specification<T>,
    private right: Specification<T>,
    private operator: 'and' | 'or'
  ) {}

  isSatisfiedBy(candidate: T): boolean {
    if (this.operator === 'and') {
      return this.left.isSatisfiedBy(candidate) && this.right.isSatisfiedBy(candidate);
    }
    return this.left.isSatisfiedBy(candidate) || this.right.isSatisfiedBy(candidate);
  }
}

// Repository with Specification
class UserRepositoryWithSpec extends BaseRepository<User> implements IUserRepository {
  async findBySpecification(spec: Specification<User>): Promise<User[]> {
    const results: User[] = [];
    for (const user of this.data.values()) {
      if (spec.isSatisfiedBy(user)) {
        results.push(user);
      }
    }
    return results;
  }

  async findByEmail(email: string): Promise<User | null> {
    const spec = new EmailSpecification(email);
    const results = await this.findBySpecification(spec);
    return results[0] || null;
  }

  async findByRole(role: string): Promise<User[]> {
    const spec = new RoleSpecification(role);
    return this.findBySpecification(spec);
  }
}

// Usage
const repo = new UserRepositoryWithSpec();
const adminSpec = new RoleSpecification('admin');
const activeAdmins = await repo.findBySpecification(adminSpec);
```

## Unit of Work Pattern with Repository

```typescript
interface UnitOfWork {
  users: IUserRepository;
  posts: IPostRepository;
  orders: IOrderRepository;
  commit(): Promise<void>;
  rollback(): Promise<void>;
}

class UnitOfWorkImpl implements UnitOfWork {
  users: IUserRepository;
  posts: IPostRepository;
  orders: IOrderRepository;

  constructor(private db: PrismaClient) {
    this.users = new UserRepositoryPrisma(db);
    this.posts = new PostRepositoryPrisma(db);
    this.orders = new OrderRepositoryPrisma(db);
  }

  async commit(): Promise<void> {
    // Begin transaction
    const transaction = await this.db.$transaction(async (tx) => {
      // All operations in repositories use the same transaction
      return { success: true };
    });
  }

  async rollback(): Promise<void> {
    // Rollback changes
  }
}

// Usage
const unitOfWork = new UnitOfWorkImpl(prisma);

try {
  const user = await unitOfWork.users.save({
    id: '1',
    name: 'John',
    email: 'john@example.com',
    createdAt: new Date()
  });

  const post = await unitOfWork.posts.save({
    id: '1',
    title: 'My Post',
    userId: user.id,
    createdAt: new Date()
  });

  await unitOfWork.commit();
} catch (error) {
  await unitOfWork.rollback();
  throw error;
}
```

## Testing with Repository

```typescript
// Mock Repository for Testing
class MockUserRepository implements IUserRepository {
  private users: Map<string, User> = new Map();

  async findById(id: string): Promise<User | null> {
    return this.users.get(id) || null;
  }

  async findAll(): Promise<User[]> {
    return Array.from(this.users.values());
  }

  async save(entity: User): Promise<User> {
    this.users.set(entity.id, entity);
    return entity;
  }

  async update(id: string, updates: Partial<User>): Promise<User> {
    const user = await this.findById(id);
    if (!user) throw new Error('User not found');
    const updated = { ...user, ...updates };
    this.users.set(id, updated);
    return updated;
  }

  async delete(id: string): Promise<boolean> {
    return this.users.delete(id);
  }

  async findByEmail(email: string): Promise<User | null> {
    for (const user of this.users.values()) {
      if (user.email === email) return user;
    }
    return null;
  }

  async findByRole(role: string): Promise<User[]> {
    return Array.from(this.users.values()).filter(u => u.role === role);
  }
}

// Test
describe('UserService', () => {
  let userService: UserService;
  let mockRepository: MockUserRepository;

  beforeEach(() => {
    mockRepository = new MockUserRepository();
    userService = new UserService(mockRepository);
  });

  test('should create a new user', async () => {
    const user = await userService.createUser('John Doe', 'john@example.com');
    expect(user.name).toBe('John Doe');
    expect(user.email).toBe('john@example.com');
  });

  test('should throw error if email already exists', async () => {
    await userService.createUser('John Doe', 'john@example.com');
    await expect(
      userService.createUser('Jane Doe', 'john@example.com')
    ).rejects.toThrow('Email already registered');
  });
});
```

## Advantages

- **Separation of Concerns:** Domain logic separated from data access
- **Testability:** Easy to mock repositories for testing
- **Flexibility:** Swap repositories without changing domain logic
- **Reusability:** Same repository used across services
- **Maintainability:** Changes to data access logic in one place
- **Database Independence:** Domain logic independent of database choice

## Disadvantages

- **Complexity:** Extra layer and abstraction
- **Boilerplate:** More code for simple operations
- **Performance:** Additional method calls and indirection
- **Learning Curve:** Developers need to understand the pattern

## Best Practices

- **Generic Repositories:** Extend base repository for common operations
- **Specific Repositories:** Create specific interfaces for domain needs
- **No Business Logic:** Repositories should only handle data access
- **Unit of Work:** Manage transactions across multiple repositories
- **Specifications:** Use specification pattern for complex queries
- **Testing:** Use mock repositories for unit testing
- **Type Safety:** Use TypeScript for compile-time checking

## Related Patterns

- **Data Mapper Pattern:** Similar but handles object/database mapping
- **Active Record:** Alternative approach where objects handle persistence
- **Unit of Work:** Coordinates repositories in transactions
- **Specification Pattern:** Complex queries using specifications

## References & Sources

- Martin Fowler - Repository Pattern: https://martinfowler.com/eaaCatalog/repository.html
- Refactoring.Guru - Repository Pattern
- Eric Evans - Domain-Driven Design
- Domain-Driven Design Community: https://dddcommunity.org/
