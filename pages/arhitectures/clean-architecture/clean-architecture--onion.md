# Onion Architecture

Onion Architecture is Jeffrey Palermo's original formulation (2008) of the dependency-inversion-based, concentric-layer style that [Clean Architecture](./clean-architecture.md) and [Hexagonal Architecture](../hexagonal-architecture/hexagonal-architecture.md) later popularized under different names. All three share the same rule — dependencies point inward, the domain model sits at the center with zero outward dependencies — Onion just draws the rings slightly differently and predates the other two names.

## Key Concepts

- **Domain Model:** The innermost ring — entities and value objects with no dependency on anything outside them
- **Domain Services:** Business logic that doesn't naturally belong to a single entity, still inside the domain ring
- **Application Services:** Orchestrate domain objects and services to fulfill use cases; this ring may define interfaces (ports) that outer rings implement
- **Outside Layer:** UI, infrastructure, database, and tests — everything that can change without touching the domain
- **No Outward References:** An inner ring never imports from an outer ring; interfaces defined inward are implemented outward (dependency inversion)

## Architecture Diagram

```
        ┌─────────────────────────────────────────┐
        │   Infrastructure / UI / Tests             │  ← outermost: swappable
        │   ┌───────────────────────────────────┐   │
        │   │      Application Services          │   │
        │   │   ┌───────────────────────────┐    │   │
        │   │   │      Domain Services        │    │   │
        │   │   │   ┌───────────────────┐     │    │   │
        │   │   │   │   Domain Model     │     │    │   │
        │   │   │   │  (Entities, VOs)   │     │    │   │
        │   │   │   └───────────────────┘     │    │   │
        │   │   └───────────────────────────┘    │   │
        │   └───────────────────────────────────┘   │
        └─────────────────────────────────────────┘
        All arrows point inward. Nothing in the center knows the outer rings exist.
```

## Onion vs. Clean vs. Hexagonal

| Aspect | Onion | Clean | Hexagonal |
|---|---|---|---|
| Origin | Jeffrey Palermo (2008) | Robert C. Martin (2012) | Alistair Cockburn (2005) |
| Core vocabulary | Rings (Domain Model, Domain Services, Application Services) | Layers (Entities, Use Cases, Interface Adapters, Frameworks) | Core, Ports, Adapters |
| Dependency rule | Inward only | Inward only | Core depends on nothing; adapters implement ports |
| Interfaces defined | Wherever an inner ring needs one, implemented outward | In the application/use-case layer | As explicit "ports" |
| Practical difference | Mostly naming — same underlying structure as Clean/Hexagonal in real codebases |

## Example: Onion-Labeled Structure

```typescript
// Domain Model (innermost ring — no imports from outside this ring)
export class Order {
  constructor(
    private readonly id: string,
    private readonly customerId: string,
    private lineItems: LineItem[],
    private status: OrderStatus = OrderStatus.Pending
  ) {}

  confirm(): void {
    if (this.status !== OrderStatus.Pending) {
      throw new Error('Only pending orders can be confirmed');
    }
    this.status = OrderStatus.Confirmed;
  }

  getTotal(): number {
    return this.lineItems.reduce((sum, item) => sum + item.getSubtotal(), 0);
  }
}

// Domain Service (still inside the domain ring, doesn't belong to one entity)
export class DiscountPolicy {
  apply(order: Order, customerTier: 'standard' | 'premium'): number {
    return customerTier === 'premium' ? order.getTotal() * 0.9 : order.getTotal();
  }
}

// Application Service (defines the port the outer ring must implement)
export interface OrderRepository {
  save(order: Order): Promise<void>;
  findById(id: string): Promise<Order | null>;
}

export class PlaceOrderService {
  constructor(
    private orderRepository: OrderRepository, // interface owned by an inner ring
    private discountPolicy: DiscountPolicy
  ) {}

  async execute(customerId: string, items: LineItem[]): Promise<Order> {
    const order = new Order(crypto.randomUUID(), customerId, items);
    order.confirm();
    await this.orderRepository.save(order); // implementation lives in the outer ring
    return order;
  }
}

// Infrastructure (outermost ring — implements the port, depends inward on the interface)
export class SqlOrderRepository implements OrderRepository {
  constructor(private db: Database) {}

  async save(order: Order): Promise<void> {
    await this.db.query('INSERT INTO orders ...', [/* ... */]);
  }

  async findById(id: string): Promise<Order | null> {
    const row = await this.db.query('SELECT * FROM orders WHERE id = $1', [id]);
    return row ? this.toOrder(row) : null;
  }

  private toOrder(row: unknown): Order { /* mapping omitted */ return {} as Order; }
}
```

## Advantages

Same as [Clean Architecture](./clean-architecture.md#advantages) — testable domain core, framework independence, flexibility to swap infrastructure — since the underlying dependency rule is identical.

## Disadvantages

Same as [Clean Architecture](./clean-architecture.md#disadvantages) — more files and abstractions than a simple layered app, a real learning curve for teams new to dependency inversion.

## When to Use

- Same criteria as Clean/Hexagonal: complex domain logic, long-lived codebase, need to swap infrastructure over time
- Teams already familiar with .NET, where Onion Architecture is a particularly common vocabulary (it originated in the .NET community)

## Related Patterns

- [Clean Architecture](./clean-architecture.md) — same dependency rule, different layer names
- [Hexagonal Architecture](../hexagonal-architecture/hexagonal-architecture.md) — same dependency rule, ports/adapters vocabulary
- [Domain-Driven Architecture](../domain-driven-architecture/domain-driven-architecture.md) — Onion's domain ring is typically modeled with DDD building blocks (entities, value objects, aggregates)

## References & Sources

- Jeffrey Palermo — "The Onion Architecture" (2008): https://jeffreypalermo.com/2008/07/the-onion-architecture-part-1/
- Robert C. Martin — "Clean Architecture: A Craftsman's Guide to Software Structure and Design" (2017)
- Alistair Cockburn — Hexagonal Architecture: https://alistair.cockburn.us/hexagonal-architecture/
