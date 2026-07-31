# Domain-Driven Architecture (DDA)

Domain-Driven Architecture is an approach to software design that emphasizes understanding and modeling the business domain. It focuses on creating a shared understanding between technical and business teams, resulting in code that directly reflects business concepts and rules.

## Key Concepts

- **Domain:** The sphere of knowledge, influence, and activity that defines the business problem
- **Ubiquitous Language:** Common vocabulary shared between developers, domain experts, and stakeholders
- **Bounded Context:** Clear boundaries within which a model is valid and applicable
- **Aggregate:** A cluster of domain objects (entities and value objects) treated as a single unit
- **Entity:** An object with identity that persists over time
- **Value Object:** Immutable object defined by its attributes, not identity
- **Repository:** Abstraction for accessing aggregates from persistent storage
- **Domain Event:** Notifications about state changes in the domain

## Core Principles

### 1. Focus on Business Domain

Model the software around the business domain, not the technical implementation.

```typescript
// ❌ Technical-focused
class UserDataManager {
  createRecord(data: any) { }
  updateRecord(id: number, data: any) { }
  deleteRecord(id: number) { }
}

// ✅ Domain-focused
class Customer {
  customerId: CustomerId;
  email: Email;
  registrationDate: Date;

  register(email: Email): void { }
  updateEmail(newEmail: Email): void { }
  deactivate(): void { }
}
```

### 2. Ubiquitous Language

Use consistent terminology that domain experts understand:

```typescript
// Domain language constants
enum OrderStatus {
  Pending = 'pending',
  Confirmed = 'confirmed',
  Shipped = 'shipped',
  Delivered = 'delivered'
}

class Order {
  private status: OrderStatus;

  confirmOrder(): void {
    // Business rule: order must be pending before confirmation
    if (this.status !== OrderStatus.Pending) {
      throw new Error('Only pending orders can be confirmed');
    }
    this.status = OrderStatus.Confirmed;
  }
}
```

### 3. Bounded Contexts

Organize code into separate contexts with clear boundaries:

```
┌─────────────────────────────────────┐
│      Order Context                  │
│  ┌─────────────────────────────┐    │
│  │ Order Aggregate             │    │
│  │ - OrderId                   │    │
│  │ - Customer (ref)            │    │
│  │ - LineItems                 │    │
│  │ - OrderStatus               │    │
│  └─────────────────────────────┘    │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│      Inventory Context              │
│  ┌─────────────────────────────┐    │
│  │ Product Aggregate           │    │
│  │ - ProductId                 │    │
│  │ - Stock                     │    │
│  │ - ReorderPoint              │    │
│  └─────────────────────────────┘    │
└─────────────────────────────────────┘
```

### 4. Aggregates

Group related entities and value objects:

```typescript
class Order implements Aggregate {
  private orderId: OrderId;
  private customerId: CustomerId;
  private lineItems: LineItem[];
  private status: OrderStatus;

  // Aggregate root enforces invariants
  addLineItem(product: Product, quantity: Quantity): void {
    if (this.status !== OrderStatus.Pending) {
      throw new Error('Cannot add items to non-pending order');
    }
    this.lineItems.push(new LineItem(product, quantity));
  }

  // Only aggregate root is persisted
  getLineItems(): ReadonlyArray<LineItem> {
    return Object.freeze([...this.lineItems]);
  }
}
```

### 5. Value Objects

Represent domain concepts with no identity:

```typescript
class Email {
  readonly value: string;

  constructor(value: string) {
    if (!this.isValid(value)) {
      throw new Error('Invalid email format');
    }
    this.value = value;
  }

  private isValid(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }

  equals(other: Email): boolean {
    return this.value === other.value;
  }
}

class Money {
  readonly amount: number;
  readonly currency: string;

  constructor(amount: number, currency: string) {
    if (amount < 0) {
      throw new Error('Amount cannot be negative');
    }
    this.amount = amount;
    this.currency = currency;
  }

  add(other: Money): Money {
    if (this.currency !== other.currency) {
      throw new Error('Cannot add different currencies');
    }
    return new Money(this.amount + other.amount, this.currency);
  }
}
```

## Architecture Patterns

### Layered Architecture with DDD

```
┌──────────────────────────────────┐
│   Presentation Layer             │ (Controllers, API endpoints)
├──────────────────────────────────┤
│   Application Layer              │ (Use cases, commands, queries)
├──────────────────────────────────┤
│   Domain Layer                   │ (Entities, value objects, domain logic)
├──────────────────────────────────┤
│   Infrastructure Layer           │ (Database, external services)
└──────────────────────────────────┘
```

### Command Query Responsibility Segregation (CQRS)

Separate read and write models:

```typescript
// Write model (Command side)
class CreateOrderCommand {
  constructor(
    public customerId: CustomerId,
    public items: Array<{ productId: ProductId; quantity: Quantity }>
  ) {}
}

class OrderCommandHandler {
  async handle(command: CreateOrderCommand): Promise<OrderId> {
    const order = Order.create(command.customerId, command.items);
    await this.orderRepository.save(order);
    return order.getId();
  }
}

// Read model (Query side)
class OrderQueryService {
  async getOrderSummary(orderId: OrderId): Promise<OrderSummaryDTO> {
    return this.readDb.query('SELECT * FROM order_summaries WHERE id = ?', [orderId]);
  }
}
```

## Folder/File Structure

Organize your project by bounded contexts, with clear separation of concerns:

```
src/
├── domains/                          # All bounded contexts
│   ├── order/                        # Order Bounded Context
│   │   ├── domain/                   # Pure domain logic
│   │   │   ├── aggregates/
│   │   │   │   ├── order.ts          # Order aggregate root
│   │   │   │   └── line-item.ts      # LineItem entity
│   │   │   ├── value-objects/
│   │   │   │   ├── order-id.ts
│   │   │   │   ├── order-status.ts
│   │   │   │   └── price.ts
│   │   │   ├── repositories/         # Repository interfaces
│   │   │   │   └── order.repository.ts
│   │   │   ├── services/             # Domain services
│   │   │   │   └── order-calculation.service.ts
│   │   │   └── events/               # Domain events
│   │   │       └── order-created.event.ts
│   │   ├── application/              # Use cases & commands
│   │   │   ├── commands/
│   │   │   │   ├── create-order.command.ts
│   │   │   │   └── create-order.handler.ts
│   │   │   ├── queries/
│   │   │   │   ├── get-order.query.ts
│   │   │   │   └── get-order.handler.ts
│   │   │   └── dto/
│   │   │       ├── create-order.dto.ts
│   │   │       └── order-summary.dto.ts
│   │   └── infrastructure/           # Database, external services
│   │       ├── repositories/         # Repository implementations
│   │       │   └── order.repository.impl.ts
│   │       ├── mappers/
│   │       │   └── order.mapper.ts
│   │       └── database/
│   │           └── order.schema.ts
│   ├── inventory/                    # Inventory Bounded Context
│   │   ├── domain/
│   │   ├── application/
│   │   └── infrastructure/
│   └── payment/                      # Payment Bounded Context
│       ├── domain/
│       ├── application/
│       └── infrastructure/
├── shared/                           # Cross-cutting concerns
│   ├── events/                       # Event bus, event handlers
│   ├── exceptions/                   # Common exceptions
│   ├── types/                        # Shared type definitions
│   └── utils/                        # Utility functions
└── main.ts                           # Application entry point
```

### File Naming Conventions

```
Value Objects:      *.value-object.ts   (email.value-object.ts)
Entities:           *.entity.ts         (customer.entity.ts)
Aggregates:         *.aggregate.ts      (order.aggregate.ts)
Repositories:       *.repository.ts     (order.repository.ts)
Services:           *.service.ts        (order-calculation.service.ts)
Commands:           *.command.ts        (create-order.command.ts)
Handlers:           *.handler.ts        (create-order.handler.ts)
Queries:            *.query.ts          (get-order.query.ts)
DTOs:               *.dto.ts            (create-order.dto.ts)
Domain Events:      *.event.ts          (order-created.event.ts)
Mappers:            *.mapper.ts         (order.mapper.ts)
Schemas/Models:     *.schema.ts         (order.schema.ts)
```

## Example: E-commerce Domain

```typescript
// Value Objects
class ProductId extends ValueObject { }
class OrderId extends ValueObject { }
class CustomerId extends ValueObject { }
class Price extends ValueObject {
  constructor(readonly value: number, readonly currency: string) {}
}

// Entity (not aggregate root)
class LineItem {
  constructor(
    readonly productId: ProductId,
    readonly quantity: number,
    readonly price: Price
  ) {}

  getTotal(): Price {
    return new Price(
      this.price.value * this.quantity,
      this.price.currency
    );
  }
}

// Aggregate Root
class Order {
  private readonly orderId: OrderId;
  private readonly customerId: CustomerId;
  private readonly lineItems: LineItem[] = [];
  private status: OrderStatus = OrderStatus.Pending;
  private createdAt: Date = new Date();

  constructor(orderId: OrderId, customerId: CustomerId) {
    this.orderId = orderId;
    this.customerId = customerId;
  }

  addLineItem(item: LineItem): void {
    if (this.status !== OrderStatus.Pending) {
      throw new Error('Cannot modify non-pending order');
    }
    this.lineItems.push(item);
  }

  confirm(): void {
    if (this.lineItems.length === 0) {
      throw new Error('Cannot confirm empty order');
    }
    this.status = OrderStatus.Confirmed;
  }

  getTotal(): Price {
    return this.lineItems.reduce(
      (total, item) => total.add(item.getTotal()),
      new Price(0, 'USD')
    );
  }
}

// Repository abstraction
interface OrderRepository {
  save(order: Order): Promise<void>;
  findById(orderId: OrderId): Promise<Order | null>;
}

// Application service
class PlaceOrderUseCase {
  constructor(private orderRepository: OrderRepository) {}

  async execute(command: PlaceOrderCommand): Promise<OrderId> {
    const order = new Order(new OrderId(), command.customerId);

    for (const item of command.items) {
      order.addLineItem(item);
    }

    order.confirm();
    await this.orderRepository.save(order);

    return order.orderId;
  }
}
```

## Advantages

- **Business Alignment:** Code reflects business concepts and rules
- **Maintainability:** Clear domain logic is easier to understand and modify
- **Scalability:** Bounded contexts allow independent team development
- **Testing:** Domain logic is isolated and easy to test
- **Communication:** Shared language reduces misunderstandings
- **Flexibility:** Business rules are explicit, not hidden in infrastructure code

## Disadvantages

- **Complexity:** Requires deeper business understanding upfront
- **Overhead:** More abstractions and layers than simple CRUD apps
- **Learning Curve:** Team needs training in DDD concepts
- **Over-engineering:** Can be overkill for simple domains
- **Maintenance:** More code to maintain and evolve

## Common Use Cases

- **E-commerce:** Complex ordering and inventory rules
- **Banking:** Strict regulatory and business rules
- **Healthcare:** Complex patient and treatment workflows
- **Insurance:** Intricate policy and claims processing
- **SaaS:** Multi-tenant systems with complex business logic

## Related Patterns

- **Domain Events:** Capture and communicate domain state changes
- **[Event Sourcing Pattern](../../patterns/event-sourcing-pattern/event-sourcing-pattern.md):** Store domain events as source of truth
- **[CQRS Pattern](../../patterns/cqrs-pattern/cqrs-pattern.md):** Separate read and write models
- **[Microservices Architecture](../microservices-architecture/microservices-architecture.md):** Each bounded context as separate service
- **[Repository Pattern](../../patterns/repository-pattern/repository-pattern.md):** Abstract data persistence
- **Service Locator:** Dependency injection for services
- **[Interpreter Architecture](../interpreter-architecture/interpreter-architecture.md):** A DSL is one way to make the ubiquitous language directly executable

## Implementation Tools & Frameworks

- **Node.js/TypeScript:** NestJS, TypeORM, Inversify
- **Java:** Spring Boot, Hibernate, Axon Framework
- **.NET:** Entity Framework, MassTransit, NServiceBus
- **Python:** SQLAlchemy, Pydantic
- **DDD Libraries:** Aggregate roots, value objects, domain events

## References & Sources

### Books
- **Eric Evans** - "Domain-Driven Design: Tackling Complexity in the Heart of Software" (2003) - The foundational DDD reference
- **Vaughn Vernon** - "Implementing Domain-Driven Design" (2013) - Practical implementation guide
- **Sam Newman** - "Building Microservices: Designing Fine-Grained Systems" (2015) - DDD in microservices context

### Articles & Guides
- Martin Fowler - Domain-Driven Design: https://martinfowler.com/bliki/DomainDrivenDesign.html
- Martin Fowler - Bounded Contexts: https://martinfowler.com/bliki/BoundedContext.html
- Microsoft - Domain-Driven Design: https://docs.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/
- DDD Community: https://dddcommunity.org/

### Related Patterns
- Aggregate Pattern
- Value Objects
- Ubiquitous Language
- Bounded Contexts
- CQRS (Command Query Responsibility Segregation)
- Event Sourcing
