# Service Pattern

The Service Pattern encapsulates business logic in dedicated service classes. Services provide reusable operations, coordinate between multiple components, and abstract complex business logic away from controllers/components. They act as the application layer between domain logic and infrastructure.

## Problem

- Business logic scattered throughout controllers/components
- Logic not reusable across routes/components
- Controller/component responsibilities mixed
- Hard to test business logic
- Complex operations not isolated
- No clear separation of concerns

## Solution

Create dedicated service classes that encapsulate business logic, handle coordination, and provide reusable operations.

## Types of Services

### 1. Application Service (Orchestration)

Coordinates operations between repositories and domain logic:

```typescript
// Application Service
class CreateOrderService {
  constructor(
    private orderRepository: OrderRepository,
    private inventoryService: InventoryService,
    private paymentService: PaymentService,
    private notificationService: NotificationService
  ) {}

  async execute(createOrderDto: CreateOrderDto): Promise<Order> {
    // 1. Validate input
    if (!createOrderDto.items || createOrderDto.items.length === 0) {
      throw new Error('Order must have items');
    }

    // 2. Check inventory
    const available = await this.inventoryService.checkAvailability(
      createOrderDto.items
    );
    if (!available) {
      throw new Error('Some items are not available');
    }

    // 3. Create order (domain logic)
    const order = Order.create(
      createOrderDto.customerId,
      createOrderDto.items,
      createOrderDto.shippingAddress
    );

    // 4. Persist order
    const savedOrder = await this.orderRepository.save(order);

    // 5. Reserve inventory
    await this.inventoryService.reserve(order.getLineItems(), savedOrder.id);

    // 6. Send notification (asynchronously)
    this.notificationService.notifyOrderCreated(savedOrder);

    return savedOrder;
  }
}

// Usage in Controller
@Controller('/orders')
export class OrderController {
  constructor(private createOrderService: CreateOrderService) {}

  @Post()
  async createOrder(@Body() dto: CreateOrderDto) {
    return this.createOrderService.execute(dto);
  }
}
```

### 2. Domain Service

Contains domain logic that doesn't belong to entities:

```typescript
// Domain Service
class PricingService {
  private taxRate = 0.1;
  private discountStrategy: DiscountStrategy;

  constructor(discountStrategy: DiscountStrategy) {
    this.discountStrategy = discountStrategy;
  }

  calculateTotal(items: OrderItem[]): Money {
    const subtotal = items.reduce((sum, item) => {
      return sum.add(item.getTotal());
    }, new Money(0, 'USD'));

    const discount = this.discountStrategy.getDiscount(subtotal);
    const afterDiscount = subtotal.multiply(1 - discount);
    const tax = afterDiscount.multiply(this.taxRate);

    return afterDiscount.add(tax);
  }

  calculateTax(amount: Money): Money {
    return amount.multiply(this.taxRate);
  }
}

// Usage
const pricingService = new PricingService(new VolumeDiscountStrategy());
const total = pricingService.calculateTotal(orderItems);
```

### 3. Infrastructure Service

Handles external service communication:

```typescript
// Infrastructure Service
class EmailService {
  private emailProvider: EmailProvider;

  constructor(emailProvider: EmailProvider) {
    this.emailProvider = emailProvider;
  }

  async sendWelcomeEmail(email: string, name: string): Promise<void> {
    const template = this.buildWelcomeTemplate(name);
    await this.emailProvider.send({
      to: email,
      subject: 'Welcome!',
      html: template
    });
  }

  async sendOrderConfirmation(email: string, order: Order): Promise<void> {
    const template = this.buildOrderTemplate(order);
    await this.emailProvider.send({
      to: email,
      subject: `Order Confirmation #${order.id}`,
      html: template
    });
  }

  private buildWelcomeTemplate(name: string): string {
    return `<h1>Welcome ${name}!</h1>`;
  }

  private buildOrderTemplate(order: Order): string {
    return `<h1>Order #${order.id}</h1>`;
  }
}

// Usage
class UserService {
  constructor(
    private userRepository: UserRepository,
    private emailService: EmailService
  ) {}

  async registerUser(email: string, name: string): Promise<User> {
    const user = User.create(email, name);
    await this.userRepository.save(user);
    await this.emailService.sendWelcomeEmail(email, name);
    return user;
  }
}
```

### 4. Stateless Service

No internal state, pure operations:

```typescript
// Stateless Service
class PasswordService {
  private saltRounds = 10;

  async hashPassword(password: string): Promise<string> {
    return bcrypt.hash(password, this.saltRounds);
  }

  async verifyPassword(password: string, hash: string): Promise<boolean> {
    return bcrypt.compare(password, hash);
  }

  validatePasswordStrength(password: string): boolean {
    const minLength = 8;
    const hasUpperCase = /[A-Z]/.test(password);
    const hasLowerCase = /[a-z]/.test(password);
    const hasNumbers = /\d/.test(password);

    return password.length >= minLength &&
           hasUpperCase &&
           hasLowerCase &&
           hasNumbers;
  }
}

// Usage
const passwordService = new PasswordService();
const isValid = passwordService.validatePasswordStrength('MyPassword123');
const hash = await passwordService.hashPassword('MyPassword123');
const matches = await passwordService.verifyPassword('MyPassword123', hash);
```

### 5. Stateful Service

Maintains state across operations:

```typescript
// Stateful Service
class ShoppingCartService {
  private carts: Map<string, Cart> = new Map();

  createCart(customerId: string): Cart {
    const cart = new Cart(customerId);
    this.carts.set(customerId, cart);
    return cart;
  }

  getCart(customerId: string): Cart {
    const cart = this.carts.get(customerId);
    if (!cart) {
      throw new Error('Cart not found');
    }
    return cart;
  }

  addItem(customerId: string, item: CartItem): void {
    const cart = this.getCart(customerId);
    cart.addItem(item);
  }

  removeItem(customerId: string, itemId: string): void {
    const cart = this.getCart(customerId);
    cart.removeItem(itemId);
  }

  checkout(customerId: string): Order {
    const cart = this.getCart(customerId);
    const order = cart.toOrder();
    this.carts.delete(customerId);
    return order;
  }
}

// Usage
const cartService = new ShoppingCartService();
cartService.createCart('user-123');
cartService.addItem('user-123', new CartItem('product-1', 2));
cartService.addItem('user-123', new CartItem('product-2', 1));
const order = cartService.checkout('user-123');
```

## Service Layer Architecture

```typescript
// Service composition for complex operations
class OrderProcessingService {
  constructor(
    private orderService: CreateOrderService,
    private paymentService: PaymentProcessingService,
    private shippingService: ShippingService,
    private notificationService: NotificationService,
    private logger: LoggerService
  ) {}

  async processOrder(order: Order): Promise<void> {
    try {
      this.logger.info(`Processing order: ${order.id}`);

      // Step 1: Process payment
      await this.paymentService.processPayment(order.id, order.getTotal());
      this.logger.info(`Payment processed: ${order.id}`);

      // Step 2: Arrange shipping
      const shippingInfo = await this.shippingService.arrangeShipping(order);
      this.logger.info(`Shipping arranged: ${order.id}`);

      // Step 3: Send notification
      await this.notificationService.sendShippingNotification(order);
      this.logger.info(`Notification sent: ${order.id}`);

    } catch (error) {
      this.logger.error(`Order processing failed: ${order.id}`, error);
      // Retry logic or compensating transaction
      throw error;
    }
  }
}

// Usage in Controller
@Controller('/orders')
export class OrderController {
  constructor(private orderProcessing: OrderProcessingService) {}

  @Post(':id/process')
  async processOrder(@Param('id') orderId: string) {
    const order = await this.getOrder(orderId);
    await this.orderProcessing.processOrder(order);
    return { success: true };
  }
}
```

## Testing Services

```typescript
// Mock dependencies
class MockOrderRepository implements OrderRepository {
  async save(order: Order): Promise<Order> {
    return order;
  }
  // ... other mock methods
}

class MockInventoryService implements InventoryService {
  async checkAvailability(items: OrderItem[]): Promise<boolean> {
    return true;
  }
  // ... other mock methods
}

// Service test
describe('CreateOrderService', () => {
  let service: CreateOrderService;
  let orderRepository: MockOrderRepository;
  let inventoryService: MockInventoryService;

  beforeEach(() => {
    orderRepository = new MockOrderRepository();
    inventoryService = new MockInventoryService();
    service = new CreateOrderService(orderRepository, inventoryService);
  });

  test('should create order successfully', async () => {
    const order = await service.execute({
      customerId: 'user-123',
      items: [{ productId: 'prod-1', quantity: 2 }]
    });

    expect(order.id).toBeDefined();
    expect(order.status).toBe('pending');
  });

  test('should throw error if no items', async () => {
    await expect(
      service.execute({
        customerId: 'user-123',
        items: []
      })
    ).rejects.toThrow('Order must have items');
  });
});
```

## Best Practices

- **Single Responsibility:** Each service has one primary purpose
- **Dependency Injection:** Inject dependencies, don't create them
- **Testability:** Services should be easy to test with mocks
- **Composition:** Compose multiple services for complex operations
- **Naming:** Use clear, action-oriented names (e.g., CreateOrderService)
- **Error Handling:** Handle and log errors appropriately
- **Stateless Preferred:** Stateless services are easier to scale
- **Separation:** Keep domain, application, and infrastructure services separate

## Service vs Controller

```typescript
// ❌ Business logic in controller
@Controller('/orders')
export class OrderController {
  @Post()
  async createOrder(@Body() dto: CreateOrderDto) {
    // Validation
    if (!dto.items || dto.items.length === 0) throw new Error('Invalid');
    
    // Business logic
    const order = new Order(dto.customerId, dto.items);
    await this.orderRepository.save(order);
    
    // Additional operations
    await this.notificationService.send(order);
    
    return order;
  }
}

// ✅ Business logic in service
@Controller('/orders')
export class OrderController {
  constructor(private createOrderService: CreateOrderService) {}

  @Post()
  async createOrder(@Body() dto: CreateOrderDto) {
    return this.createOrderService.execute(dto);
  }
}
```

## Related Patterns

- **Dependency Injection:** Inject dependencies into services
- **Repository Pattern:** Services use repositories for data access
- **Factory Pattern:** Services can use factories to create objects
- **Facade Pattern:** Service acts as facade to complex operations

## References & Sources

- Domain-Driven Design - Eric Evans
- Clean Architecture - Robert C. Martin
- NestJS Services: https://docs.nestjs.com/providers
- Angular Services: https://angular.io/guide/architecture-services
