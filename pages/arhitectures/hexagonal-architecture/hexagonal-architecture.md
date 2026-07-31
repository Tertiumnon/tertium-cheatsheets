# Hexagonal Architecture (Ports & Adapters)

Hexagonal Architecture, also known as Ports and Adapters pattern, is an architectural style that isolates the application core (domain logic) from external concerns like databases, frameworks, and UI. It enables building systems that are testable, maintainable, and independent of external technologies.

## Key Concepts

- **Core/Domain:** Business logic isolated from external dependencies
- **Ports:** Interfaces that define contracts for communication
- **Adapters:** Implementations that connect domain to external systems
- **Inbound (Driving) Adapters:** Controllers, CLI, APIs that drive the application
- **Outbound (Driven) Adapters:** Database, external services, email providers
- **Dependency Inversion:** Domain depends on abstractions, not concretions
- **Technology Independence:** Swap implementations without changing core logic

## Architecture Diagram

```
                    ┌─────────────────────────────────┐
                    │    EXTERNAL WORLD               │
                    │  (Databases, APIs, UI, etc)    │
                    └──────────┬──────────────────────┘
                               │
                ┌──────────────┴──────────────┐
                │      ADAPTER LAYER          │
        ┌───────▼────────┐          ┌────────▼────────┐
        │   Input        │          │   Output        │
        │   Adapters     │          │   Adapters      │
        │ (Controllers)  │          │ (Repositories)  │
        └───────┬────────┘          └────────┬────────┘
                │                            │
        ┌───────▼────────────────────────────▼────────┐
        │           PORT LAYER                        │
        │  (Interfaces/Contracts)                    │
        └───────┬────────────────────────────┬───────┘
                │                            │
        ┌───────▼────────────────────────────▼────────┐
        │                                              │
        │        APPLICATION CORE (DOMAIN)            │
        │   - Business Logic                          │
        │   - Entities                                │
        │   - Use Cases                               │
        │   - Domain Rules                            │
        │                                              │
        └──────────────────────────────────────────────┘
```

## Project Structure

```
src/
├── domain/                           # Application Core
│   ├── entities/
│   │   ├── User.ts
│   │   ├── Order.ts
│   │   └── Product.ts
│   ├── value-objects/
│   │   ├── Email.ts
│   │   ├── Money.ts
│   │   └── OrderStatus.ts
│   ├── use-cases/                    # Application services
│   │   ├── CreateOrderUseCase.ts
│   │   ├── UpdateUserUseCase.ts
│   │   └── GetOrderDetailsUseCase.ts
│   └── exceptions/
│       ├── DomainException.ts
│       ├── UserNotFoundException.ts
│       └── InvalidOrderException.ts
├── ports/                            # Interfaces (Contracts)
│   ├── inbound/                      # Driving ports
│   │   ├── OrderController.port.ts
│   │   └── UserController.port.ts
│   └── outbound/                     # Driven ports
│       ├── UserRepository.port.ts
│       ├── OrderRepository.port.ts
│       ├── EmailService.port.ts
│       ├── PaymentGateway.port.ts
│       └── ExternalNotificationService.port.ts
├── adapters/
│   ├── inbound/
│   │   ├── http/
│   │   │   ├── controllers/
│   │   │   │   ├── OrderController.ts
│   │   │   │   └── UserController.ts
│   │   │   ├── middleware/
│   │   │   └── routes/
│   │   └── cli/
│   │       └── commands/
│   └── outbound/
│       ├── persistence/
│       │   ├── repositories/
│       │   │   ├── UserRepository.ts
│       │   │   └── OrderRepository.ts
│       │   └── database/
│       │       ├── connection.ts
│       │       └── migrations/
│       └── external/
│           ├── services/
│           │   ├── EmailServiceAdapter.ts
│           │   ├── PaymentGatewayAdapter.ts
│           │   └── NotificationServiceAdapter.ts
│           └── config/
├── application/
│   └── registry.ts                   # Dependency injection
└── main.ts                           # Entry point
```

## Domain Layer (Core)

### Entity

```typescript
// domain/entities/Order.ts
export enum OrderStatus {
  Pending = 'pending',
  Confirmed = 'confirmed',
  Shipped = 'shipped',
  Delivered = 'delivered',
  Cancelled = 'cancelled'
}

export class Order {
  private readonly id: string;
  private readonly customerId: string;
  private lineItems: LineItem[];
  private status: OrderStatus;
  private total: Money;
  private readonly createdAt: Date;

  constructor(
    id: string,
    customerId: string,
    lineItems: LineItem[],
    total: Money
  ) {
    this.id = id;
    this.customerId = customerId;
    this.lineItems = lineItems;
    this.total = total;
    this.status = OrderStatus.Pending;
    this.createdAt = new Date();

    this.validate();
  }

  private validate(): void {
    if (!this.customerId) {
      throw new InvalidOrderException('Customer ID is required');
    }
    if (this.lineItems.length === 0) {
      throw new InvalidOrderException('Order must have at least one item');
    }
  }

  getId(): string { return this.id; }
  getStatus(): OrderStatus { return this.status; }
  getCustomerId(): string { return this.customerId; }
  getTotal(): Money { return this.total; }
  getLineItems(): readonly LineItem[] { return Object.freeze([...this.lineItems]); }

  confirm(): void {
    if (this.status !== OrderStatus.Pending) {
      throw new InvalidOrderException('Only pending orders can be confirmed');
    }
    this.status = OrderStatus.Confirmed;
  }

  ship(): void {
    if (this.status !== OrderStatus.Confirmed) {
      throw new InvalidOrderException('Only confirmed orders can be shipped');
    }
    this.status = OrderStatus.Shipped;
  }

  cancel(): void {
    if ([OrderStatus.Shipped, OrderStatus.Delivered].includes(this.status)) {
      throw new InvalidOrderException('Cannot cancel shipped or delivered order');
    }
    this.status = OrderStatus.Cancelled;
  }

  getOrderValue(): Money {
    return this.lineItems.reduce(
      (total, item) => total.add(item.getTotal()),
      new Money(0, 'USD')
    );
  }
}
```

### Value Object

```typescript
// domain/value-objects/Money.ts
export class Money {
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

  multiply(factor: number): Money {
    return new Money(this.amount * factor, this.currency);
  }

  equals(other: Money): boolean {
    return this.amount === other.amount && this.currency === other.currency;
  }

  toString(): string {
    return `${this.currency} ${this.amount}`;
  }
}
```

### Use Case

```typescript
// domain/use-cases/CreateOrderUseCase.ts
import { Order } from '../entities/Order';
import { OrderRepository } from '../../ports/outbound/OrderRepository.port';
import { UserRepository } from '../../ports/outbound/UserRepository.port';
import { InvalidOrderException } from '../exceptions/InvalidOrderException';

export interface CreateOrderInput {
  customerId: string;
  items: Array<{ productId: string; quantity: number; price: number }>;
}

export class CreateOrderUseCase {
  constructor(
    private orderRepository: OrderRepository,
    private userRepository: UserRepository
  ) {}

  async execute(input: CreateOrderInput): Promise<Order> {
    // Validate user exists
    const user = await this.userRepository.findById(input.customerId);
    if (!user) {
      throw new InvalidOrderException('User not found');
    }

    // Create order
    const lineItems = input.items.map(item =>
      new LineItem(item.productId, item.quantity, new Money(item.price, 'USD'))
    );

    const total = lineItems.reduce(
      (sum, item) => sum.add(item.getTotal()),
      new Money(0, 'USD')
    );

    const order = new Order(
      generateId(),
      input.customerId,
      lineItems,
      total
    );

    // Persist
    await this.orderRepository.save(order);

    return order;
  }
}
```

## Ports (Interfaces)

### Inbound Port (Driving)

```typescript
// ports/inbound/OrderController.port.ts
import { Order } from '../../domain/entities/Order';

export interface OrderController {
  createOrder(request: CreateOrderRequest): Promise<Order>;
  getOrder(id: string): Promise<Order>;
  updateOrderStatus(id: string, status: string): Promise<Order>;
  cancelOrder(id: string): Promise<void>;
}

export interface CreateOrderRequest {
  customerId: string;
  items: Array<{ productId: string; quantity: number }>;
}
```

### Outbound Port (Driven)

```typescript
// ports/outbound/OrderRepository.port.ts
import { Order } from '../../domain/entities/Order';

export interface OrderRepository {
  save(order: Order): Promise<void>;
  findById(id: string): Promise<Order | null>;
  findByCustomerId(customerId: string): Promise<Order[]>;
  update(order: Order): Promise<void>;
  delete(id: string): Promise<void>;
}

// ports/outbound/EmailService.port.ts
export interface EmailService {
  sendOrderConfirmation(email: string, orderId: string): Promise<void>;
  sendOrderShipped(email: string, orderId: string, trackingNumber: string): Promise<void>;
  sendOrderCancellation(email: string, orderId: string): Promise<void>;
}

// ports/outbound/PaymentGateway.port.ts
export interface PaymentGateway {
  processPayment(amount: number, currency: string, token: string): Promise<string>;
  refund(transactionId: string): Promise<void>;
  verifyPayment(transactionId: string): Promise<boolean>;
}
```

## Adapters

### Inbound Adapter (HTTP Controller)

```typescript
// adapters/inbound/http/controllers/OrderController.ts
import { Router, Request, Response } from 'express';
import { OrderController as OrderControllerPort } from '../../ports/inbound/OrderController.port';
import { CreateOrderUseCase } from '../../domain/use-cases/CreateOrderUseCase';

export class OrderController implements OrderControllerPort {
  constructor(
    private createOrderUseCase: CreateOrderUseCase
  ) {}

  setupRoutes(): Router {
    const router = Router();

    router.post('/orders', async (req: Request, res: Response) => {
      try {
        const order = await this.createOrder(req.body);
        res.status(201).json(order);
      } catch (error) {
        res.status(400).json({ error: error.message });
      }
    });

    router.get('/orders/:id', async (req: Request, res: Response) => {
      try {
        const order = await this.getOrder(req.params.id);
        res.json(order);
      } catch (error) {
        res.status(404).json({ error: 'Order not found' });
      }
    });

    return router;
  }

  async createOrder(request: any) {
    return this.createOrderUseCase.execute({
      customerId: request.customerId,
      items: request.items
    });
  }

  async getOrder(id: string) {
    // Implementation
  }

  async updateOrderStatus(id: string, status: string) {
    // Implementation
  }

  async cancelOrder(id: string) {
    // Implementation
  }
}
```

### Outbound Adapter (Database)

```typescript
// adapters/outbound/persistence/repositories/OrderRepository.ts
import { Order } from '../../../domain/entities/Order';
import { OrderRepository as OrderRepositoryPort } from '../../../ports/outbound/OrderRepository.port';
import { Database } from '../database/connection';

export class OrderRepository implements OrderRepositoryPort {
  constructor(private db: Database) {}

  async save(order: Order): Promise<void> {
    await this.db.query(
      `INSERT INTO orders (id, customer_id, status, total, created_at)
       VALUES ($1, $2, $3, $4, $5)`,
      [order.getId(), order.getCustomerId(), order.getStatus(), order.getTotal(), order.createdAt]
    );

    for (const item of order.getLineItems()) {
      await this.db.query(
        `INSERT INTO order_items (order_id, product_id, quantity, price)
         VALUES ($1, $2, $3, $4)`,
        [order.getId(), item.productId, item.quantity, item.price]
      );
    }
  }

  async findById(id: string): Promise<Order | null> {
    const result = await this.db.query(
      'SELECT * FROM orders WHERE id = $1',
      [id]
    );

    if (result.rows.length === 0) return null;

    return this.mapToOrder(result.rows[0]);
  }

  async update(order: Order): Promise<void> {
    await this.db.query(
      `UPDATE orders SET status = $1 WHERE id = $2`,
      [order.getStatus(), order.getId()]
    );
  }

  async delete(id: string): Promise<void> {
    await this.db.query('DELETE FROM orders WHERE id = $1', [id]);
  }

  async findByCustomerId(customerId: string): Promise<Order[]> {
    const result = await this.db.query(
      'SELECT * FROM orders WHERE customer_id = $1 ORDER BY created_at DESC',
      [customerId]
    );

    return result.rows.map(row => this.mapToOrder(row));
  }

  private mapToOrder(row: any): Order {
    // Map database row to Order entity
    return new Order(row.id, row.customer_id, [], new Money(row.total, 'USD'));
  }
}
```

### Outbound Adapter (External Service)

```typescript
// adapters/outbound/external/services/EmailServiceAdapter.ts
import { EmailService as EmailServicePort } from '../../../ports/outbound/EmailService.port';
import axios from 'axios';

export class EmailServiceAdapter implements EmailServicePort {
  private client = axios.create({
    baseURL: process.env.EMAIL_SERVICE_URL,
    headers: {
      'Authorization': `Bearer ${process.env.EMAIL_SERVICE_KEY}`
    }
  });

  async sendOrderConfirmation(email: string, orderId: string): Promise<void> {
    await this.client.post('/send', {
      to: email,
      template: 'order_confirmation',
      data: { orderId }
    });
  }

  async sendOrderShipped(
    email: string,
    orderId: string,
    trackingNumber: string
  ): Promise<void> {
    await this.client.post('/send', {
      to: email,
      template: 'order_shipped',
      data: { orderId, trackingNumber }
    });
  }

  async sendOrderCancellation(email: string, orderId: string): Promise<void> {
    await this.client.post('/send', {
      to: email,
      template: 'order_cancelled',
      data: { orderId }
    });
  }
}
```

## Dependency Injection / Registry

```typescript
// application/registry.ts
import { Database } from '../adapters/outbound/persistence/database/connection';
import { OrderRepository } from '../adapters/outbound/persistence/repositories/OrderRepository';
import { UserRepository } from '../adapters/outbound/persistence/repositories/UserRepository';
import { EmailServiceAdapter } from '../adapters/outbound/external/services/EmailServiceAdapter';
import { PaymentGatewayAdapter } from '../adapters/outbound/external/services/PaymentGatewayAdapter';
import { CreateOrderUseCase } from '../domain/use-cases/CreateOrderUseCase';
import { OrderController } from '../adapters/inbound/http/controllers/OrderController';

export class ApplicationRegistry {
  private static instance: ApplicationRegistry;
  private db: Database;

  private constructor() {
    this.db = new Database();
  }

  static getInstance(): ApplicationRegistry {
    if (!ApplicationRegistry.instance) {
      ApplicationRegistry.instance = new ApplicationRegistry();
    }
    return ApplicationRegistry.instance;
  }

  // Repositories
  getOrderRepository(): OrderRepository {
    return new OrderRepository(this.db);
  }

  getUserRepository(): UserRepository {
    return new UserRepository(this.db);
  }

  // External Services
  getEmailService(): EmailServiceAdapter {
    return new EmailServiceAdapter();
  }

  getPaymentGateway(): PaymentGatewayAdapter {
    return new PaymentGatewayAdapter();
  }

  // Use Cases
  getCreateOrderUseCase(): CreateOrderUseCase {
    return new CreateOrderUseCase(
      this.getOrderRepository(),
      this.getUserRepository()
    );
  }

  // Controllers
  getOrderController(): OrderController {
    return new OrderController(
      this.getCreateOrderUseCase()
    );
  }
}
```

## Testing

Easy to test because domain logic is isolated:

```typescript
// __tests__/domain/use-cases/CreateOrderUseCase.test.ts
import { CreateOrderUseCase } from '../../../domain/use-cases/CreateOrderUseCase';
import { OrderRepository } from '../../../ports/outbound/OrderRepository.port';
import { UserRepository } from '../../../ports/outbound/UserRepository.port';

class MockOrderRepository implements OrderRepository {
  async save(order: any): Promise<void> { }
  async findById(id: string): Promise<any> { return null; }
  async update(order: any): Promise<void> { }
  async delete(id: string): Promise<void> { }
  async findByCustomerId(customerId: string): Promise<any[]> { return []; }
}

class MockUserRepository implements UserRepository {
  async findById(id: string): Promise<any> {
    return { id, name: 'Test User', email: 'test@example.com' };
  }
}

describe('CreateOrderUseCase', () => {
  let useCase: CreateOrderUseCase;
  let orderRepository: MockOrderRepository;
  let userRepository: MockUserRepository;

  beforeEach(() => {
    orderRepository = new MockOrderRepository();
    userRepository = new MockUserRepository();
    useCase = new CreateOrderUseCase(orderRepository, userRepository);
  });

  test('should create a valid order', async () => {
    const input = {
      customerId: '123',
      items: [
        { productId: 'P1', quantity: 2, price: 100 }
      ]
    };

    const order = await useCase.execute(input);

    expect(order.getId()).toBeDefined();
    expect(order.getCustomerId()).toBe('123');
    expect(order.getStatus()).toBe('pending');
  });

  test('should throw error for non-existent user', async () => {
    userRepository.findById = jest.fn().mockResolvedValue(null);

    const input = {
      customerId: 'INVALID',
      items: [{ productId: 'P1', quantity: 1, price: 100 }]
    };

    await expect(useCase.execute(input)).rejects.toThrow('User not found');
  });
});
```

## Advantages

- **Testability:** Domain logic is framework-agnostic and easy to test
- **Flexibility:** Swap adapters without changing domain logic
- **Maintainability:** Clear separation of concerns
- **Reusability:** Domain logic can be reused in different contexts (CLI, API, batch jobs)
- **Independence:** Domain doesn't depend on external frameworks
- **Clarity:** Business rules are explicit and centralized
- **Scalability:** Easy to add new adapters

## Disadvantages

- **Complexity:** More files and layers for simple applications
- **Learning Curve:** Team needs to understand the pattern
- **Boilerplate:** Requires interfaces for every port
- **Overhead:** Additional abstraction layers can slow performance

## Best Practices

- **Domain Focus:** Keep domain pure, all dependencies are injected
- **Thin Controllers:** Controllers should only transform requests/responses
- **Port Clarity:** Ports should represent clear contracts
- **Adapter Simplicity:** Adapters should be thin wrappers
- **Exception Handling:** Domain exceptions vs infrastructure exceptions
- **Testing:** Test domain logic independently of adapters
- **Documentation:** Document port contracts clearly

## File Naming Conventions

```
Entities:           *.ts                (Order.ts, User.ts)
Value Objects:      *.ts                (Money.ts, Email.ts)
Use Cases:          *UseCase.ts         (CreateOrderUseCase.ts)
Exceptions:         *Exception.ts       (OrderNotFoundException.ts)
Port Interfaces:    *.port.ts           (OrderRepository.port.ts)
Adapters:           *Adapter.ts         (EmailServiceAdapter.ts)
Repositories:       *Repository.ts      (OrderRepository.ts)
Controllers:        *Controller.ts      (OrderController.ts)
```

## References & Sources

### Books
- **Alistair Cockburn** - "Hexagonal Architecture" - Original author
- **Robert C. Martin** - "Clean Architecture: A Craftsman's Guide to Software Structure and Design"
- **Eric Evans** - "Domain-Driven Design"

### Articles & Guides
- Hexagonal Architecture: https://alistair.cockburn.us/hexagonal-architecture/
- Fernando Cejas - Clean Architecture: https://fernandocejas.com/2018/architecture/
- Uncle Bob - Clean Architecture: https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html

### Related Patterns
- [Clean Architecture](../clean-architecture/clean-architecture.md) and [Onion Architecture](../clean-architecture/clean-architecture--onion.md) — same dependency-inversion rule, different vocabulary
- [Layered Architecture](../layered-architecture/layered-architecture.md) — the un-inverted style this pattern replaces layer-to-layer calls in
- [Domain-Driven Architecture](../domain-driven-architecture/domain-driven-architecture.md)
- [Dependency Injection Pattern](../../patterns/dependency-injection-pattern/dependency-injection.md)
- [Repository Pattern](../../patterns/repository-pattern/repository-pattern.md)
- [Adapter Pattern](../../patterns/adapter-pattern/adapter-pattern.md)

### Technologies
- **Inversion of Control (IoC) Containers:** Spring, NestJS, Inversify
- **Databases:** PostgreSQL, MongoDB
- **Testing:** Jest, Mocha, RSpec
