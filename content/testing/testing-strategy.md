# Testing Strategy

Testing is the second layer of defense against AI-generated code errors. A comprehensive testing strategy with 80%+ code coverage acts as an automatic filter for logical errors that TypeScript strict mode might miss.

## Why Testing Matters with AI

AI can write TypeScript-correct code that is logically wrong:

```typescript
// ✅ TypeScript is happy (correct types)
// ❌ Logic is wrong (missing division by 100)
function discountPrice(price: number, percent: number): number {
  return price - (price * percent); // Should be: (price * percent / 100)
}

// Test catches this:
it('should calculate 10% discount correctly', () => {
  expect(discountPrice(100, 10)).toBe(90); // FAIL!
});
```

## Testing Pyramid

```
        ▲
       /│\
      / │ \         E2E Tests (5%)
     /  │  \        Full application flow
    /   │   \
   /    │    \      Integration Tests (15%)
  /     │     \     Component interaction
 /      │      \    Database, external APIs
/───────┼───────\
        │        ─── Unit Tests (80%)
        │            Pure functions
        │            Single class methods
        │            No external dependencies
        │
    Testing Pyramid
```

## Level 1: Unit Tests (80% of coverage)

Test individual functions and methods in isolation, **without external dependencies**.

### Unit Testing Best Practices

```typescript
// ✅ GOOD: Test pure function
describe('PricingService.calculateDiscount', () => {
  it('should calculate 10% discount correctly', () => {
    const service = new PricingService();
    expect(service.calculateDiscount(100, 10)).toBe(90);
  });

  it('should handle edge case: 0% discount', () => {
    const service = new PricingService();
    expect(service.calculateDiscount(100, 0)).toBe(100);
  });

  it('should handle edge case: 100% discount', () => {
    const service = new PricingService();
    expect(service.calculateDiscount(100, 100)).toBe(0);
  });

  it('should reject negative discount', () => {
    const service = new PricingService();
    expect(() => service.calculateDiscount(100, -10)).toThrow();
  });

  it('should reject negative price', () => {
    const service = new PricingService();
    expect(() => service.calculateDiscount(-100, 10)).toThrow();
  });
});

// ✅ GOOD: Test with mocked dependencies
describe('OrderService.createOrder', () => {
  let orderService: OrderService;
  let mockUserRepository: MockUserRepository;
  let mockOrderRepository: MockOrderRepository;

  beforeEach(() => {
    mockUserRepository = new MockUserRepository();
    mockOrderRepository = new MockOrderRepository();
    orderService = new OrderService(mockUserRepository, mockOrderRepository);
  });

  it('should create order for valid user', async () => {
    mockUserRepository.setUser({ id: '1', name: 'John' });
    const order = await orderService.createOrder('1', []);
    expect(order.userId).toBe('1');
  });

  it('should throw error if user not found', async () => {
    await expect(orderService.createOrder('invalid', [])).rejects.toThrow();
  });

  it('should throw error if order has no items', async () => {
    mockUserRepository.setUser({ id: '1', name: 'John' });
    await expect(orderService.createOrder('1', [])).rejects.toThrow();
  });

  it('should save order to repository', async () => {
    mockUserRepository.setUser({ id: '1', name: 'John' });
    await orderService.createOrder('1', [{ productId: '1', qty: 1 }]);
    expect(mockOrderRepository.saved).toBe(true);
  });
});

// ❌ BAD: Testing with real database (this is integration test, not unit test)
describe('OrderService - NOT A UNIT TEST', () => {
  it('should save to real database', async () => {
    const db = new Database(); // Real connection
    const service = new OrderService(db);
    await service.createOrder('1', []);
    // This is slow, fragile, and not a unit test
  });
});
```

### Mock Examples

```typescript
// Mock User Repository
class MockUserRepository implements UserRepository {
  private user: User | null = null;

  setUser(user: User | null) {
    this.user = user;
  }

  async findById(id: string): Promise<User | null> {
    return this.user?.id === id ? this.user : null;
  }
}

// Mock Order Repository
class MockOrderRepository implements OrderRepository {
  public saved = false;
  private order: Order | null = null;

  async save(order: Order): Promise<void> {
    this.saved = true;
    this.order = order;
  }

  async findById(id: string): Promise<Order | null> {
    return this.order?.id === id ? this.order : null;
  }
}
```

## Level 2: Integration Tests (15% of coverage)

Test how **multiple components work together**, including real databases or external services.

```typescript
describe('OrderService Integration - Database', () => {
  let orderService: OrderService;
  let db: TestDatabase;

  beforeAll(async () => {
    db = new TestDatabase(); // Real test database
    await db.connect();
  });

  afterAll(async () => {
    await db.disconnect();
  });

  beforeEach(async () => {
    await db.clear();
    const userRepository = new UserRepositoryImpl(db);
    const orderRepository = new OrderRepositoryImpl(db);
    orderService = new OrderService(userRepository, orderRepository);
  });

  it('should create order and persist to database', async () => {
    // Setup: Create user in database
    await db.users.insert({ id: '1', name: 'John', email: 'john@example.com' });

    // Execute: Create order
    const order = await orderService.createOrder('1', [
      { productId: 'prod-1', quantity: 2 }
    ]);

    // Assert: Verify database state
    const savedOrder = await db.orders.findById(order.id);
    expect(savedOrder).toBeTruthy();
    expect(savedOrder.status).toBe('pending');
  });

  it('should cascade delete when user is deleted', async () => {
    await db.users.insert({ id: '1', name: 'John' });
    const order = await orderService.createOrder('1', []);

    await db.users.delete('1');

    const savedOrder = await db.orders.findById(order.id);
    expect(savedOrder).toBeNull(); // Cascade delete
  });
});

// HTTP Integration Test
describe('OrderController Integration - HTTP', () => {
  let app: Application;
  let request: SuperTest<Test>;

  beforeAll(async () => {
    app = createApp();
    request = supertest(app);
  });

  it('should create order via POST /orders', async () => {
    const response = await request
      .post('/orders')
      .set('Authorization', 'Bearer token')
      .send({
        customerId: '1',
        items: [{ productId: 'prod-1', quantity: 2 }]
      });

    expect(response.status).toBe(201);
    expect(response.body.id).toBeDefined();
  });

  it('should return 400 if items are empty', async () => {
    const response = await request
      .post('/orders')
      .set('Authorization', 'Bearer token')
      .send({
        customerId: '1',
        items: []
      });

    expect(response.status).toBe(400);
  });

  it('should return 401 if not authenticated', async () => {
    const response = await request
      .post('/orders')
      .send({
        customerId: '1',
        items: [{ productId: 'prod-1', quantity: 2 }]
      });

    expect(response.status).toBe(401);
  });
});
```

## Level 3: E2E Tests (5% of coverage)

Test complete user workflows through the entire application.

```typescript
describe('Order Creation E2E', () => {
  let browser: Browser;
  let page: Page;

  beforeAll(async () => {
    browser = await puppeteer.launch();
  });

  afterAll(async () => {
    await browser.close();
  });

  beforeEach(async () => {
    page = await browser.newPage();
    await page.goto('http://localhost:3000');
  });

  it('should complete order flow: login -> browse -> checkout', async () => {
    // Step 1: Login
    await page.click('[data-testid="login-btn"]');
    await page.type('[data-testid="email-input"]', 'user@example.com');
    await page.type('[data-testid="password-input"]', 'password');
    await page.click('[data-testid="submit-btn"]');
    await page.waitForNavigation();

    // Step 2: Browse products
    await page.click('[data-testid="products-link"]');
    await page.click('[data-testid="product-1"]');
    await page.click('[data-testid="add-to-cart"]');

    // Step 3: Checkout
    await page.click('[data-testid="cart-link"]');
    await page.click('[data-testid="checkout-btn"]');
    await page.click('[data-testid="place-order-btn"]');

    // Verify
    await page.waitForSelector('[data-testid="order-confirmation"]');
    const text = await page.$eval('[data-testid="order-id"]', el => el.textContent);
    expect(text).toContain('Order #');
  });
});
```

## Test Coverage Analysis

```bash
# Run tests with coverage report
npm run test -- --coverage

# Output example:
# ────────────────────────────────────────────────────────
# File                    | % Stmts | % Branch | % Funcs |
# ────────────────────────────────────────────────────────
# OrderService.ts         | 95.2%   | 92.1%    | 100%    |
# UserService.ts          | 87.3%    | 81.0%    | 90%     |
# PricingService.ts       | 100%     | 100%     | 100%    |
# ────────────────────────────────────────────────────────
# TOTAL                   | 87.5%    | 84.6%    | 96.7%   |
```

## Testing Tools Comparison

| Tool | Use Case | Speed | Setup |
|------|----------|-------|-------|
| **Vitest** | Unit tests (JavaScript) | ⚡⚡⚡ Fast | Simple |
| **Jest** | Unit + Integration | ⚡⚡ Medium | Moderate |
| **Mocha** | General testing | ⚡⚡ Medium | Manual |
| **Cypress** | E2E testing | 🐢 Slow | Medium |
| **Playwright** | E2E + API testing | ⚡⚡ Fast | Simple |
| **Angular Testing** | Angular components | ⚡⚡ Medium | Built-in |
| **React Testing Library** | React components | ⚡⚡⚡ Fast | Simple |

## Test-Driven Development (TDD)

Red-Green-Refactor cycle:

```typescript
// 1. RED: Write test that fails
it('should calculate tax correctly', () => {
  const service = new TaxService();
  expect(service.calculateTax(100, 0.1)).toBe(10);
  // Test fails: method doesn't exist yet
});

// 2. GREEN: Write minimal implementation to pass
class TaxService {
  calculateTax(amount: number, rate: number): number {
    return amount * rate;
  }
}
// Test passes!

// 3. REFACTOR: Improve implementation
class TaxService {
  calculateTax(amount: number, rate: number): number {
    if (amount < 0 || rate < 0 || rate > 1) {
      throw new Error('Invalid input');
    }
    return Math.round(amount * rate * 100) / 100;
  }
}
// Test still passes, but better implementation
```

## Testing Checklist for AI-Generated Code

When reviewing AI-generated code, ensure:

- ✅ All edge cases tested (empty, null, negative values)
- ✅ Error conditions tested
- ✅ Boundary values tested
- ✅ Race conditions tested (for async code)
- ✅ Database state tested (for integration tests)
- ✅ Authentication/authorization tested
- ✅ 80%+ code coverage achieved
- ✅ Tests actually verify behavior (not just run)

## Best Practices

- **Test Naming:** Use descriptive names: `should_returnDiscount_when_percentIsValid`
- **Arrange-Act-Assert:** Setup → Execute → Verify
- **DRY Tests:** Use `beforeEach` and fixtures to reduce duplication
- **Mock External:** Always mock databases, APIs, file systems
- **Test Independence:** Tests should run in any order
- **Snapshot Testing:** Use for UI components, not business logic
- **Performance Tests:** Test critical paths for performance regressions

## References & Sources

- Jest Documentation: https://jestjs.io/
- Vitest: https://vitest.dev/
- Testing Library: https://testing-library.com/
- Cypress: https://www.cypress.io/
- Google Testing Blog: https://testing.googleblog.com/
- "Working Effectively with Legacy Code" - Michael Feathers
