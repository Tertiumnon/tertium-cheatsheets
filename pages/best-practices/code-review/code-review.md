# Code Review Guidelines

Code Review is the final human-controlled layer of defense against AI-generated code errors. It catches logical mistakes, architectural violations, and edge cases that TypeScript and tests might miss.

## Why Code Review is Critical for AI-Generated Code

AI can produce code that passes TypeScript checks and tests but contains:
- **Architectural violations:** Breaking feature boundaries, circular dependencies
- **Logical errors:** Off-by-one errors, wrong comparison operators, inverted conditions
- **Performance issues:** N+1 queries, unnecessary loops, missing caches
- **Security vulnerabilities:** SQL injection risks, authentication bypasses, data exposure
- **Edge cases:** Race conditions, timeout handling, retry logic failures

```typescript
// ✅ Passes TypeScript strict mode
// ✅ Passes tests
// ❌ Architectural violation: breaks DDD domain boundary
export class UserService {
  createOrder(userId: string, items: OrderItem[]) {
    // ERROR: UserService should not know about orders!
    // This belongs in OrderService
    const order = new Order(userId, items);
    this.db.save(order);
    return order;
  }
}
```

## Code Review Layers

### Layer 1: Architecture Compliance

Verify code follows the project's architecture.

```typescript
// CHECKLIST:
- ✅ Is this code in the correct folder/module?
- ✅ Does it respect layer boundaries (domain, application, infrastructure)?
- ✅ Does it follow the feature-based structure (if using it)?
- ✅ Are cross-module dependencies through facades/event-bus only?
- ✅ No imports from internal files of other features?

// Example: Feature-based check
// ✅ GOOD:
import { UserService } from '../auth/auth.facade';

// ❌ BAD (internal implementation detail):
import { TokenService } from '../auth/services/token.service';
```

### Layer 2: Logic Correctness

Verify the algorithm and logic are correct.

```typescript
// Example AI mistake (passes tests, fails in production)
function calculateDiscount(price: number, discountPercent: number): number {
  return price - (price * discountPercent); // WRONG: forgot to divide by 100!
  // Should be: price - (price * discountPercent / 100)
}

// CHECKLIST:
- ✅ Are all edge cases handled correctly?
- ✅ Are boundary values (0, negative, max) handled?
- ✅ Is the algorithm correct for the stated requirement?
- ✅ Are assumptions documented and validated?
- ✅ Is error handling appropriate?

// Example: Check error paths
function getUser(id: string): User {
  const user = findUser(id);
  return user; // WRONG: returns undefined if not found!
  // Should throw or return null explicitly
}
```

### Layer 3: Performance

Verify no performance regressions.

```typescript
// Example: N+1 query problem
async function getAllUsersWithOrders(): Promise<User[]> {
  const users = await db.query('SELECT * FROM users');
  for (const user of users) {
    user.orders = await db.query('SELECT * FROM orders WHERE userId = ?', user.id);
    // WRONG: N+1 queries! (1 for users + N for each user's orders)
  }
  return users;
}

// CHECKLIST:
- ✅ Are there N+1 query problems?
- ✅ Are expensive operations (DB, API calls) minimized?
- ✅ Is caching used where appropriate?
- ✅ Are loops optimized (not doing work inside tight loops)?
- ✅ Are unnecessary computations avoided?
```

### Layer 4: Security

Verify no security vulnerabilities.

```typescript
// Example: SQL injection vulnerability
async function findUserByEmail(email: string): Promise<User | null> {
  return db.query(`SELECT * FROM users WHERE email = '${email}'`);
  // WRONG: SQL injection vulnerability!
  // User can pass: ' OR '1'='1
}

// ✅ GOOD:
async function findUserByEmail(email: string): Promise<User | null> {
  return db.query('SELECT * FROM users WHERE email = ?', [email]);
}

// CHECKLIST:
- ✅ No SQL injection vulnerabilities (use parameterized queries)
- ✅ No hardcoded secrets or API keys
- ✅ No XSS vulnerabilities (user input sanitized)
- ✅ Authentication/authorization properly enforced
- ✅ Sensitive data not logged or exposed in errors
- ✅ No privilege escalation opportunities
- ✅ Input validation performed at system boundary
```

### Layer 5: Error Handling

Verify errors are handled appropriately.

```typescript
// Example: Poor error handling
async function processPayment(amount: Money): Promise<string> {
  const response = await paymentGateway.charge(amount);
  return response.transactionId;
  // WRONG: No error handling! What if charge fails?
}

// ✅ GOOD:
async function processPayment(amount: Money): Promise<string> {
  try {
    const response = await paymentGateway.charge(amount);
    if (!response.success) {
      throw new PaymentFailedError(response.error);
    }
    return response.transactionId;
  } catch (error) {
    if (error instanceof NetworkError) {
      throw new PaymentTemporarilyUnavailableError();
    }
    throw new PaymentFailedError(error.message);
  }
}

// CHECKLIST:
- ✅ All async operations have error handlers
- ✅ Errors are properly caught and transformed
- ✅ User-facing errors are clear and non-technical
- ✅ Internal errors are logged for debugging
- ✅ Errors don't expose sensitive information
- ✅ Retry logic for transient errors
```

### Layer 6: Test Coverage and Quality

Verify tests are thorough and meaningful.

```typescript
// Example: Test that doesn't verify behavior
it('should call the repository', () => {
  service.createUser(user);
  expect(mockRepository.save).toHaveBeenCalled();
  // WRONG: Test doesn't verify the result or user properties!
});

// ✅ GOOD:
it('should save user with correct properties', () => {
  const user = { name: 'John', email: 'john@example.com' };
  const result = service.createUser(user);
  expect(mockRepository.save).toHaveBeenCalledWith(expect.objectContaining({
    name: 'John',
    email: 'john@example.com'
  }));
  expect(result.id).toBeDefined();
});

// CHECKLIST:
- ✅ Tests verify behavior, not just that functions are called
- ✅ Edge cases are tested (empty, null, boundary values)
- ✅ Error paths are tested
- ✅ Test names describe the behavior being tested
- ✅ Tests are independent and can run in any order
- ✅ No hardcoded timeouts or flaky assertions
```

### Layer 7: Code Style and Consistency

Verify code follows project conventions.

```typescript
// CHECKLIST:
- ✅ Variable/function names are clear and descriptive
- ✅ No magic numbers (constants extracted)
- ✅ Comments explain WHY, not WHAT
- ✅ Functions are not too long (< 50 lines ideally)
- ✅ Classes have single responsibility
- ✅ No commented-out code
- ✅ Consistent indentation and formatting
- ✅ No console.log statements (use logger)
- ✅ No unused imports or variables
```

### Layer 8: Documentation

Verify code is properly documented.

```typescript
// Example: Poorly documented function
function process(input: any) {
  // WRONG: No documentation, unclear what it does
  return input.map(x => x * 2).filter(x => x > 10);
}

// ✅ GOOD:
/**
 * Filters transactions by minimum amount.
 * @param transactions - Array of transaction objects
 * @param minAmount - Minimum transaction amount in cents
 * @returns Transactions with amount >= minAmount, doubled
 */
function filterTransactionsByAmount(
  transactions: Transaction[],
  minAmount: number
): Transaction[] {
  return transactions
    .map(t => ({ ...t, amount: t.amount * 2 }))
    .filter(t => t.amount > minAmount);
}

// CHECKLIST:
- ✅ Public functions have JSDoc comments
- ✅ Complex algorithms have explanatory comments
- ✅ Non-obvious decisions are documented
- ✅ README updated if adding new features
- ✅ Type definitions are clear
```

## Code Review Checklist for AI-Generated Code

### Pre-Review (Automated)
- ✅ TypeScript strict mode passes
- ✅ Linter passes (ESLint, Prettier)
- ✅ Tests pass (unit, integration, E2E)
- ✅ Coverage is 80%+ (for new code)
- ✅ Build succeeds

### Architecture Review
- ✅ Code is in correct location per architecture
- ✅ No circular dependencies
- ✅ Feature boundaries respected
- ✅ Dependency Injection used (dependencies not hardcoded)
- ✅ No tight coupling to frameworks

### Logic Review
- ✅ Algorithm is correct for stated requirement
- ✅ Edge cases handled (empty, null, zero, negative, max)
- ✅ Boundary conditions tested
- ✅ Off-by-one errors checked
- ✅ Boolean logic correct (not inverted conditions)
- ✅ Comparison operators correct (=== vs ==, < vs <=)

### Performance Review
- ✅ No N+1 query problems
- ✅ Expensive operations minimized
- ✅ Caching used for repeated operations
- ✅ Loops don't do unnecessary work
- ✅ No blocking operations in async code
- ✅ Memory leaks prevented (listeners unsubscribed)

### Security Review
- ✅ No SQL injection (parameterized queries)
- ✅ No hardcoded secrets
- ✅ No XSS vulnerabilities
- ✅ Authentication/authorization enforced
- ✅ Input validation performed
- ✅ Sensitive data not logged
- ✅ Error messages don't leak information

### Error Handling Review
- ✅ All async operations have error handlers
- ✅ Errors properly caught and logged
- ✅ User-facing errors are clear
- ✅ Retry logic for transient failures
- ✅ Timeout handling for long operations
- ✅ No silent failures

### Testing Review
- ✅ Tests verify behavior, not implementation
- ✅ Edge cases tested
- ✅ Error paths tested
- ✅ Test names describe behavior
- ✅ No hardcoded timeouts
- ✅ Tests are deterministic

### Style and Consistency Review
- ✅ Naming is clear and consistent
- ✅ No magic numbers
- ✅ Functions are reasonable size
- ✅ Single responsibility principle
- ✅ No commented-out code
- ✅ Proper logging, no console.log
- ✅ No unused imports/variables

## Common AI-Generated Code Issues

### Issue 1: Incorrect Logic

```typescript
// AI writes:
function isEligibleForDiscount(age: number): boolean {
  return age > 18 && age < 65; // WRONG: retirees 65+ not eligible
}

// Should be:
function isEligibleForDiscount(age: number): boolean {
  return age >= 18 && age < 65;
}
```

### Issue 2: Missing Null Checks

```typescript
// AI writes:
function getOrderTotal(order: Order): number {
  return order.items.reduce((sum, item) => sum + item.price, 0);
  // WRONG: order.items could be null/undefined
}

// Should be:
function getOrderTotal(order: Order): number {
  if (!order || !order.items) {
    throw new Error('Invalid order');
  }
  return order.items.reduce((sum, item) => sum + item.price, 0);
}
```

### Issue 3: Hardcoded Values

```typescript
// AI writes:
async function getUserSubscription(userId: string) {
  return db.query('SELECT * FROM subscriptions WHERE userId = ? AND status = "active"', [userId]);
  // WRONG: Status value hardcoded, should be constant
}

// Should be:
const SUBSCRIPTION_STATUS = {
  ACTIVE: 'active',
  INACTIVE: 'inactive'
} as const;

async function getUserSubscription(userId: string) {
  return db.query(
    'SELECT * FROM subscriptions WHERE userId = ? AND status = ?',
    [userId, SUBSCRIPTION_STATUS.ACTIVE]
  );
}
```

### Issue 4: No Error Handling

```typescript
// AI writes:
async function fetchUser(id: string) {
  const response = await fetch(`/api/users/${id}`);
  return response.json();
  // WRONG: No error handling!
}

// Should be:
async function fetchUser(id: string) {
  const response = await fetch(`/api/users/${id}`);
  if (!response.ok) {
    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
  }
  return response.json();
}
```

### Issue 5: Performance Problems

```typescript
// AI writes:
function processOrders(orders: Order[]) {
  return orders.map(order => {
    order.total = calculateTotal(order); // Expensive calculation in loop
    return order;
  });
}

// Should be:
function processOrders(orders: Order[]) {
  const totals = new Map();
  return orders.map(order => ({
    ...order,
    total: totals.get(order.id) || calculateTotal(order)
  }));
}
```

## Code Review Process for AI-Generated Code

### Step 1: Automated Checks (CI/CD)
```bash
npm run lint
npm run type-check
npm run test
npm run test:coverage
```

### Step 2: Architecture Validation
```bash
npm run depcruise # Check dependency rules
```

### Step 3: Manual Code Review
1. Read the git diff
2. Check architecture compliance
3. Verify logic correctness
4. Review error handling
5. Check for security issues
6. Verify test coverage

### Step 4: Request Changes or Approve
- **Request Changes:** Specific improvements needed
- **Comment:** Questions or suggestions
- **Approve:** Code is ready to merge

## Code Review Comments for AI-Generated Code

### Good Comments
```typescript
// Missing null check - what if order doesn't exist?
async function getOrder(id: string): Promise<Order> {
  return db.findById(id); // ERROR: Need null check
}

// This breaks the OrderService responsibility - should be in OrderService
class UserService {
  createOrder(userId: string) {
    // ERROR: This violates feature boundaries
  }
}

// This could be an N+1 query in production with large datasets
for (const user of users) {
  user.orders = await db.find({ userId: user.id }); // WARN: N+1 potential
}

// Test doesn't verify the actual behavior
it('should create order', () => {
  service.createOrder(dto);
  expect(mockRepo.save).toHaveBeenCalled(); // WARN: Doesn't verify order properties
});
```

### What NOT to Say
```typescript
// ❌ "This code is bad"
// ✅ "This could fail if order doesn't exist. Consider: if (!order) throw new Error(...)"

// ❌ "I would write this differently"
// ✅ "This accesses index without checking length. Use optional chaining or bounds check"

// ❌ "This violates best practices"
// ✅ "This hardcodes the database URL. Use environment variable instead"
```

## When to Approve vs Request Changes

### Approve If:
- ✅ Code follows architecture
- ✅ Logic is correct
- ✅ No security issues
- ✅ Error handling is proper
- ✅ Tests are thorough
- ✅ Performance is acceptable
- ✅ Code is readable

### Request Changes If:
- ❌ Logic is incorrect or unclear
- ❌ Architecture violated
- ❌ Missing error handling
- ❌ Security vulnerability
- ❌ Poor test coverage
- ❌ Performance issue
- ❌ Unacceptable code style

## Code Review for Different Roles

### Senior Developer Reviewer
- Focus on architecture and design patterns
- Check for technical debt accumulation
- Verify scalability implications
- Review security decisions

### Team Lead Reviewer
- Verify alignment with project goals
- Check team coding standards
- Ensure documentation is complete
- Monitor overall code quality trends

### QA Reviewer
- Test scenario completeness
- Edge case coverage
- Error path testing
- Performance testing

## Three-Layer Quality Defense System

```
Layer 1: TypeScript Strict Mode
├─ Catches type errors
├─ Prevents null/undefined issues
└─ Automatic enforcement

Layer 2: Testing (80%+)
├─ Catches logical errors
├─ Verifies edge cases
└─ Automated validation

Layer 3: Code Review (Human)
├─ Catches architectural issues
├─ Verifies security
├─ Checks performance
└─ Manual validation
```

## Related Topics

- Testing Strategy (Layer 2 filter)
- TypeScript Strict Mode (Layer 1 filter)
- Architecture Patterns
- Security Best Practices

## References & Sources

- Google Code Review Developer Guide: https://google.github.io/eng-practices/review/
- Best Kept Secrets of Peer Code Review - SmartBear
- Code Review from the Command Line: https://github.com/features/code-review
- Security Code Review: https://owasp.org/www-community/Code_Review_Guide
