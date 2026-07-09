# Clean Architecture

Clean Architecture is a design approach that organizes code into concentric circles of layers, each with increasing abstraction. The core principle is that business logic should be independent of external frameworks, databases, UI, and web servers. This creates a system that is testable, maintainable, and flexible to change.

## Key Concepts

- **Dependency Inversion:** Inner layers don't depend on outer layers
- **Layers:** Clear separation from core business logic to external concerns
- **Testability:** Business logic can be tested without frameworks or UI
- **Independence:** Business rules don't depend on delivery mechanism (web, mobile, CLI)
- **Framework Agnostic:** Business logic is isolated from specific frameworks

## Architecture Layers

```
┌─────────────────────────────────────┐
│     Presentation Layer              │ (UI, Controllers, Routes)
├─────────────────────────────────────┤
│     Application Layer               │ (Use Cases, Application Services)
├─────────────────────────────────────┤
│     Domain Layer                    │ (Entities, Business Rules)
├─────────────────────────────────────┤
│     Infrastructure Layer            │ (Database, APIs, Frameworks)
└─────────────────────────────────────┘
```

## Project Structure

```
src/
├── domain/                           # Core business logic (no dependencies)
│   ├── entities/
│   │   ├── User.ts
│   │   ├── Order.ts
│   │   └── Product.ts
│   ├── use-cases/
│   │   ├── CreateOrderUseCase.ts
│   │   ├── UpdateUserUseCase.ts
│   │   └── GetOrderDetailsUseCase.ts
│   ├── services/
│   │   ├── PricingService.ts
│   │   └── ValidationService.ts
│   └── exceptions/
│       ├── BusinessException.ts
│       ├── InvalidOrderException.ts
│       └── UserNotFoundException.ts
├── application/                      # Use case implementation
│   ├── dto/
│   │   ├── CreateOrderDTO.ts
│   │   ├── UpdateUserDTO.ts
│   │   └── ResponseDTO.ts
│   ├── services/
│   │   ├── OrderApplicationService.ts
│   │   └── UserApplicationService.ts
│   └── repositories/                 # Abstractions
│       ├── UserRepository.interface.ts
│       ├── OrderRepository.interface.ts
│       └── PaymentRepository.interface.ts
├── infrastructure/                   # External concerns
│   ├── persistence/
│   │   ├── UserRepositoryImpl.ts
│   │   ├── OrderRepositoryImpl.ts
│   │   └── Database.ts
│   ├── external/
│   │   ├── PaymentGateway.ts
│   │   ├── EmailService.ts
│   │   └── SmsService.ts
│   └── config/
│       └── Container.ts              # Dependency injection
├── presentation/                     # UI/API Layer
│   ├── controllers/
│   │   ├── OrderController.ts
│   │   └── UserController.ts
│   ├── routes/
│   │   ├── orderRoutes.ts
│   │   └── userRoutes.ts
│   ├── middleware/
│   │   ├── AuthMiddleware.ts
│   │   └── ErrorHandler.ts
│   └── viewmodels/
│       └── UserViewModel.ts
└── main.ts
```

## Layer Responsibilities

### Domain Layer (Core Business Logic)

Pure business logic with **zero external dependencies**:

```typescript
// domain/entities/Order.ts
export class Order {
  private readonly id: string;
  private readonly customerId: string;
  private lineItems: LineItem[];
  private status: OrderStatus;
  private total: Money;

  constructor(id: string, customerId: string, lineItems: LineItem[]) {
    if (!customerId) throw new Error('Customer ID required');
    if (lineItems.length === 0) throw new Error('Order must have items');
    
    this.id = id;
    this.customerId = customerId;
    this.lineItems = lineItems;
    this.status = OrderStatus.Pending;
    this.total = this.calculateTotal();
  }

  private calculateTotal(): Money {
    return this.lineItems.reduce(
      (sum, item) => sum.add(item.getTotal()),
      new Money(0, 'USD')
    );
  }

  confirm(): void {
    if (this.status !== OrderStatus.Pending) {
      throw new Error('Only pending orders can be confirmed');
    }
    this.status = OrderStatus.Confirmed;
  }

  getStatus(): OrderStatus { return this.status; }
  getTotal(): Money { return this.total; }
}

// domain/use-cases/CreateOrderUseCase.ts
export class CreateOrderUseCase {
  async execute(input: CreateOrderInput): Order {
    // Pure business logic - no database, no frameworks
    const order = new Order(
      input.id,
      input.customerId,
      input.lineItems
    );
    return order;
  }
}
```

### Application Layer (Use Case Implementation)

Orchestrates domain logic and coordinates with infrastructure:

```typescript
// application/services/OrderApplicationService.ts
export class OrderApplicationService {
  constructor(
    private orderRepository: OrderRepository,
    private userRepository: UserRepository,
    private createOrderUseCase: CreateOrderUseCase
  ) {}

  async createOrder(dto: CreateOrderDTO): Promise<OrderResponseDTO> {
    // 1. Validate user exists (infrastructure)
    const user = await this.userRepository.findById(dto.customerId);
    if (!user) throw new UserNotFoundException();

    // 2. Execute use case (domain)
    const order = await this.createOrderUseCase.execute({
      id: generateId(),
      customerId: dto.customerId,
      lineItems: dto.items
    });

    // 3. Persist (infrastructure)
    await this.orderRepository.save(order);

    // 4. Return DTO
    return OrderResponseDTO.fromEntity(order);
  }
}
```

### Infrastructure Layer (External Concerns)

Implements interfaces defined in application layer:

```typescript
// infrastructure/persistence/UserRepositoryImpl.ts
export class UserRepositoryImpl implements UserRepository {
  constructor(private db: Database) {}

  async findById(id: string): Promise<User | null> {
    const row = await this.db.query('SELECT * FROM users WHERE id = $1', [id]);
    return row ? User.fromRow(row) : null;
  }

  async save(user: User): Promise<void> {
    await this.db.query(
      'INSERT INTO users (id, email, name) VALUES ($1, $2, $3)',
      [user.id, user.email, user.name]
    );
  }
}

// infrastructure/external/PaymentGateway.ts
export class PaymentGateway {
  constructor(private httpClient: HttpClient) {}

  async processPayment(amount: Money, token: string): Promise<string> {
    const response = await this.httpClient.post('/process', {
      amount: amount.value,
      currency: amount.currency,
      token
    });
    return response.transactionId;
  }
}
```

### Presentation Layer (UI/API)

Controllers and routes that use application services:

```typescript
// presentation/controllers/OrderController.ts
@Controller('/orders')
export class OrderController {
  constructor(private orderService: OrderApplicationService) {}

  @Post()
  async create(@Body() dto: CreateOrderDTO): Promise<OrderResponseDTO> {
    return this.orderService.createOrder(dto);
  }

  @Get(':id')
  async getById(@Param('id') id: string): Promise<OrderResponseDTO> {
    return this.orderService.getOrder(id);
  }
}
```

## Dependency Flow

```
Presentation Layer
      ↓ uses
Application Layer
      ↓ uses
Domain Layer (no dependencies)
      ↓ implements
Infrastructure Layer
```

**Key Rule:** Inner layers never know about outer layers. Domain logic is completely independent.

## Testing Strategy

Domain layer is testable without any infrastructure:

```typescript
describe('Order Entity', () => {
  test('should create order with valid data', () => {
    const order = new Order('1', 'customer-1', [lineItem]);
    expect(order.getStatus()).toBe(OrderStatus.Pending);
  });

  test('should throw error if no customer ID', () => {
    expect(() => new Order('1', '', [lineItem])).toThrow();
  });
});

describe('OrderApplicationService', () => {
  let service: OrderApplicationService;
  let mockOrderRepository: MockOrderRepository;
  let mockUserRepository: MockUserRepository;

  beforeEach(() => {
    mockOrderRepository = new MockOrderRepository();
    mockUserRepository = new MockUserRepository();
    service = new OrderApplicationService(
      mockOrderRepository,
      mockUserRepository,
      createOrderUseCase
    );
  });

  test('should save order to repository', async () => {
    await service.createOrder(createOrderDTO);
    expect(mockOrderRepository.saved).toBeTruthy();
  });
});
```

## Advantages

- **Testability:** Business logic tested without frameworks
- **Flexibility:** Easy to swap databases, frameworks, APIs
- **Maintainability:** Clear separation and dependencies
- **Independence:** Business rules not coupled to implementation
- **Longevity:** Code survives technology changes
- **Scalability:** Easy to understand and extend

## Disadvantages

- **Complexity:** More layers and abstractions
- **Overhead:** More files and structure required
- **Learning Curve:** Team needs to understand layered approach
- **Boilerplate:** More DTOs, interfaces, and mapping code

## Best Practices

- **Dependency Inversion:** Depend on abstractions, not concretions
- **Entity Purity:** Entities contain only business rules
- **Use Cases:** Each use case is a separate class
- **DTOs:** Convert between layers using Data Transfer Objects
- **Dependency Injection:** Inject all dependencies
- **Testing:** Unit test domain layer without infrastructure
- **Interfaces:** Define contracts in application layer

## When to Use

- Large, long-lived applications
- Team of multiple developers
- Business logic is complex
- Need to change frameworks/databases
- High test coverage is critical
- Maintenance is priority

## Common Mistakes

- **Mixing Layers:** Putting database code in entities
- **Fat Controllers:** Business logic in presentation layer
- **Circular Dependencies:** Outer layers depending on inner
- **Skipping DTOs:** Using entities across layer boundaries
- **Over-Engineering:** Too many layers for simple apps

## Related Patterns

- **Hexagonal Architecture:** Similar but focuses on ports/adapters
- **Domain-Driven Design:** Similar focus on domain isolation
- **Repository Pattern:** Used in infrastructure layer
- **Dependency Injection:** Essential for clean architecture

## References & Sources

- Robert C. Martin - Clean Architecture: https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html
- Clean Code: A Handbook of Agile Software Craftsmanship
- Clean Architecture: A Craftsman's Guide to Software Structure and Design
- Microsoft Architecture - Clean Architecture: https://docs.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/common-web-application-architectures#clean-architecture
