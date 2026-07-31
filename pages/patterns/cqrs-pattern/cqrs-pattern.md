# CQRS Pattern (Command Query Responsibility Segregation)

CQRS separates the operations that change state (commands) from the operations that read state (queries) into distinct models, sometimes backed by entirely separate data stores. Instead of one model serving both reads and writes, each side is optimized for its own job — commands enforce invariants and business rules, queries optimize for fast, denormalized reads.

## Problem

- A single model serving both writes and reads gets awkward once either side has real complexity
- Read workloads (dashboards, listings, reports) need denormalized, query-optimized shapes that don't match a rich write model
- Write and read workloads scale differently — reads are usually much higher volume than writes
- Enforcing invariants on write gets tangled with formatting/projection concerns needed for read

## Solution

Split into a **command side** that validates input, enforces business rules, and persists changes, and a **query side** that serves pre-shaped, denormalized read models — kept in sync via direct writes or, in the distributed form, via published events.

## Structure

```
        Commands                              Queries
           │                                     │
           ▼                                     ▼
   ┌───────────────┐                    ┌────────────────┐
   │ Command Handler│                    │  Query Handler  │
   └───────┬───────┘                    └────────┬────────┘
           │ validates + writes                    │ reads
           ▼                                     ▼
   ┌───────────────┐   (sync write or    ┌────────────────┐
   │  Write Model    │───published event──▶│   Read Model    │
   │ (normalized)    │      / projection   │ (denormalized) │
   └───────────────┘                    └────────────────┘
```

## Basic Implementation (Shared Database)

```typescript
// Command side — enforces invariants, the only path allowed to change state
interface CreateOrderCommand {
  customerId: string;
  items: Array<{ productId: string; quantity: number; unitPrice: number }>;
}

class CreateOrderCommandHandler {
  constructor(private orderRepository: OrderWriteRepository) {}

  async handle(command: CreateOrderCommand): Promise<string> {
    if (command.items.length === 0) {
      throw new Error('Order must contain at least one item');
    }
    const orderId = crypto.randomUUID();
    const total = command.items.reduce((sum, i) => sum + i.unitPrice * i.quantity, 0);

    await this.orderRepository.insert({
      id: orderId,
      customerId: command.customerId,
      items: command.items,
      total,
      status: 'pending'
    });

    return orderId;
  }
}

// Query side — shaped exactly for what the UI needs, never used to mutate state
interface OrderSummary {
  orderId: string;
  customerName: string;
  itemCount: number;
  total: number;
  status: string;
}

class GetOrderSummaryQueryHandler {
  constructor(private readDb: ReadDatabase) {}

  async handle(orderId: string): Promise<OrderSummary> {
    return this.readDb.queryOne(
      `SELECT o.id as "orderId", c.name as "customerName",
              jsonb_array_length(o.items) as "itemCount", o.total, o.status
       FROM orders o JOIN customers c ON c.id = o.customer_id
       WHERE o.id = $1`,
      [orderId]
    );
  }
}
```

## Advanced: Separate Read/Write Stores via Projection

```typescript
// The write side only knows about the write model and publishes what happened —
// it has no idea a read model even exists.
class OrderWriteService {
  constructor(private writeRepo: OrderWriteRepository, private eventBus: EventBus) {}

  async createOrder(command: CreateOrderCommand): Promise<string> {
    const orderId = crypto.randomUUID();
    await this.writeRepo.insert({ id: orderId, ...command, status: 'pending' });

    await this.eventBus.publish('order.created', {
      orderId,
      customerId: command.customerId,
      items: command.items,
      total: command.items.reduce((sum, i) => sum + i.unitPrice * i.quantity, 0)
    });

    return orderId;
  }
}

// A projector rebuilds the denormalized read model from events — independently scalable,
// independently optimized (e.g. Elasticsearch, a reporting DB, or a materialized view)
class OrderSummaryProjector {
  constructor(private readDb: ReadDatabase) {
    // subscribes at startup
  }

  async onOrderCreated(event: { orderId: string; customerId: string; total: number }): Promise<void> {
    await this.readDb.upsert('order_summaries', {
      orderId: event.orderId,
      customerId: event.customerId,
      total: event.total,
      status: 'pending'
    });
  }
}
```

## Advantages

- **Independent Scaling:** Read and write paths scale according to their own load profile
- **Optimized Models:** Each side is shaped for its actual job instead of compromising between the two
- **Simpler Queries:** Denormalized read models avoid complex joins at query time
- **Clear Write Boundaries:** All state changes funnel through explicit, validated commands

## Disadvantages

- **Eventual Consistency:** When read/write stores are separate, a read immediately after a write may not reflect it yet
- **Increased Complexity:** Two models (and potentially two data stores) to build, test, and keep in sync
- **Duplication:** Read models duplicate data that already exists in the write model
- **Overkill for Simple CRUD:** Most screens don't need this split; forcing it everywhere adds cost without benefit

## When to Use

- Read and write workloads have very different volume, shape, or performance requirements
- Complex domain logic on write, combined with rich reporting/dashboard needs on read
- Systems already using [Event Sourcing](../event-sourcing-pattern/event-sourcing-pattern.md), where CQRS is a near-natural fit
- High-read, low-write systems where read-model denormalization meaningfully improves latency

## Best Practices

- **Start with One Database:** Split read/write models logically first; only split the physical store when you actually need to
- **Make Commands Explicit:** Name commands as intents (`CreateOrder`, not `SaveOrder`) and validate them fully before applying
- **Version Read Models Independently:** A read model can be rebuilt/reshaped without touching the write side
- **Monitor Projection Lag:** If read/write stores are separate, track how far behind the read model is

## Common Mistakes

- **Splitting Everywhere:** Applying CQRS to simple CRUD screens that never needed the separation
- **Leaking Write Model into Reads:** Querying the write model directly for reporting, defeating the purpose of a dedicated read model
- **No Consistency Guarantees Communicated:** Not telling the UI/client that a read might briefly lag behind a write
- **Coupling Command and Query Handlers:** Sharing one class for both, reintroducing the exact coupling CQRS is meant to remove

## Related Patterns

- [Event Sourcing Pattern](../event-sourcing-pattern/event-sourcing-pattern.md) — commonly paired with CQRS; events become the mechanism that syncs write and read models
- [Domain-Driven Architecture](../../arhitectures/domain-driven-architecture/domain-driven-architecture.md) — CQRS is frequently applied at the aggregate/bounded-context level
- [Event-Driven Architecture](../../arhitectures/event-driven-architecture/event-driven-architecture.md) — the transport CQRS uses to propagate writes into read projections
- [Repository Pattern](../repository-pattern/repository-pattern.md) — typical abstraction for both the write and read stores

## References & Sources

- Greg Young — CQRS Documents: https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf
- Martin Fowler — CQRS: https://martinfowler.com/bliki/CQRS.html
- Microsoft — CQRS Pattern: https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs
