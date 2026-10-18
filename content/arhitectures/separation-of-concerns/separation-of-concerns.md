# Separation of Concerns (SoC)

Separation of Concerns divides a system into distinct sections, each addressing exactly one concern, so that a change to one concern requires little or no change to the others. It's less a specific structure than a principle every other architectural style applies in its own way — layers separate concerns horizontally, features separate them vertically, hexagonal separates domain from infrastructure, and aspect-oriented techniques separate cross-cutting concerns like logging or auth from business logic.

## Key Concepts

- **Concern:** A distinct piece of functionality or responsibility (validation, persistence, formatting, authorization)
- **Cohesion:** How closely related the responsibilities within one module are — high cohesion means a module does one thing well
- **Coupling:** How much one module depends on the internals of another — low coupling means concerns can change independently
- **Cross-Cutting Concern:** A concern that would otherwise be scattered across many unrelated modules (logging, auth, caching, metrics)
- **Modularity:** The practical unit SoC is applied at — files, classes, layers, services, or features

## Techniques

```
Horizontal separation (by layer)          Vertical separation (by feature)
┌─────────────────────────┐               ┌────────┬────────┬────────┐
│      Presentation        │               │  Auth  │ Orders │ Users  │
├─────────────────────────┤               │        │        │        │
│      Business Logic      │               │  UI    │  UI    │  UI    │
├─────────────────────────┤               │  Logic │  Logic │  Logic │
│      Persistence         │               │  Data  │  Data  │  Data  │
└─────────────────────────┘               └────────┴────────┴────────┘

Aspect separation (cross-cutting, orthogonal to both)
┌──────────────────────────────────────────────────┐
│  Logging · Authorization · Caching · Metrics       │  ← applied across every layer/feature
│  without living inside any of them                 │
└──────────────────────────────────────────────────┘
```

## Example: Mixed Concerns vs. Separated

```typescript
// ❌ Concerns mixed: validation, auth, logging, business logic, and persistence
// all live in one handler with no clear ownership of any single responsibility
app.post('/orders', async (req, res) => {
  if (!req.headers.authorization) return res.status(401).send('Unauthorized');
  if (!req.body.items || req.body.items.length === 0) {
    return res.status(400).send('Order must have items');
  }
  console.log(`Creating order for ${req.body.customerId}`);
  const total = req.body.items.reduce((sum: number, i: any) => sum + i.price * i.qty, 0);
  const order = { id: crypto.randomUUID(), ...req.body, total };
  await db.query('INSERT INTO orders VALUES ($1, $2, $3)', [order.id, order.customerId, order.total]);
  res.status(201).json(order);
});
```

```typescript
// ✅ Each concern owns exactly one responsibility
const router = Router();

router.post(
  '/orders',
  authMiddleware,                 // concern: authorization
  validateCreateOrder,            // concern: validation
  loggingMiddleware('order.create'), // concern: observability
  async (req: Request, res: Response) => {
    const order = await orderService.createOrder(req.body); // concern: business logic
    res.status(201).json(order);
  }
);

class OrderService {
  constructor(private orderRepository: OrderRepository, private pricingService: PricingService) {}

  async createOrder(dto: CreateOrderDto): Promise<Order> {
    const total = this.pricingService.calculateTotal(dto.items); // concern: pricing
    return this.orderRepository.save({ ...dto, total });          // concern: persistence
  }
}
```

## Example: Cross-Cutting Concerns via Decorators

```typescript
// Logging and caching are applied without polluting the business method itself
function Logged(): MethodDecorator {
  return (target, propertyKey, descriptor: PropertyDescriptor) => {
    const original = descriptor.value;
    descriptor.value = async function (...args: unknown[]) {
      console.log(`Calling ${String(propertyKey)} with`, args);
      const result = await original.apply(this, args);
      console.log(`${String(propertyKey)} returned`, result);
      return result;
    };
  };
}

function Cached(ttlMs: number): MethodDecorator {
  const cache = new Map<string, { value: unknown; expiresAt: number }>();
  return (target, propertyKey, descriptor: PropertyDescriptor) => {
    const original = descriptor.value;
    descriptor.value = async function (...args: unknown[]) {
      const key = `${String(propertyKey)}:${JSON.stringify(args)}`;
      const cached = cache.get(key);
      if (cached && cached.expiresAt > Date.now()) return cached.value;

      const result = await original.apply(this, args);
      cache.set(key, { value: result, expiresAt: Date.now() + ttlMs });
      return result;
    };
  };
}

class ProductCatalog {
  @Logged()
  @Cached(60_000)
  async getProduct(id: string): Promise<Product> {
    return productRepository.findById(id); // pure business logic, nothing else
  }
}
```

## Advantages

- **Independent Change:** A concern can be modified, tested, or replaced without touching unrelated code
- **Reusability:** A well-isolated concern (logging, auth, caching) can be applied anywhere it's needed
- **Easier Reasoning:** Each module has one job, making it easier to understand in isolation
- **Parallel Work:** Different people/teams can own different concerns simultaneously

## Disadvantages

- **Indirection:** More files, layers, and middleware to trace through than one big function
- **Over-Separation:** Splitting too aggressively creates ceremony without real benefit for trivial logic
- **Cross-Concern Coordination:** Some problems genuinely span concerns (a validation error that also needs to be logged and localized), and forcing a clean split can be artificial
- **Discoverability:** Related logic scattered across many small, "single-purpose" files can be harder to find than logic kept together

## When to Use

- Any codebase past a trivial size — SoC is the default, not the exception
- Systems with genuine cross-cutting concerns (auth, logging, caching, metrics, i18n)
- Codebases maintained by multiple people/teams who need to work without stepping on each other
- Any point where a single function or class is doing more than one distinguishable job

## Best Practices

- **One Reason to Change:** If a module changes for two unrelated reasons, split it (Single Responsibility Principle applied at any granularity)
- **Extract Cross-Cutting Concerns:** Use middleware, decorators, or interceptors instead of repeating logging/auth/caching inline
- **Name by Concern, Not by Type:** `OrderValidator`, not `Helper1` — the name should say which concern it owns
- **Don't Over-Split:** A three-line function doesn't need its own layer; separate concerns at the point where reuse or independent change actually matters

## Common Mistakes

- **God Objects:** A single class that absorbs validation, business rules, persistence, and formatting because splitting "felt like overkill" early on
- **Leaky Layers:** A presentation-layer concern (HTTP status codes) leaking into business logic that should be transport-agnostic
- **Duplicated Cross-Cutting Logic:** Copy-pasting the same logging/auth code into every handler instead of extracting it once
- **Confusing SoC with File Count:** Splitting one concern across many tiny files doesn't separate concerns — it just spreads the same concern out

## Related Patterns

- [Layered Architecture](../layered-architecture/layered-architecture.md) — horizontal application of SoC
- [Feature-Based Architecture](../feature-based-architecture/feature-based-architecture.md) — vertical application of SoC
- [Clean Architecture](../clean-architecture/clean-architecture.md) — SoC combined with dependency inversion
- [Decorator Pattern](../../patterns/decorator-pattern/decorator-pattern.md) — a common mechanism for isolating cross-cutting concerns

## References & Sources

- Edsger W. Dijkstra — "On the Role of Scientific Thought" (1974), the paper that coined the term
- Robert C. Martin — "Clean Code: A Handbook of Agile Software Craftsmanship" (2008), Single Responsibility Principle
- Robert C. Martin — "Clean Architecture: A Craftsman's Guide to Software Structure and Design" (2017)
