# Data-Centric Architecture

Data-Centric Architecture puts a shared, persistent data store at the center of the system, with independent components reading from and writing to that store rather than communicating directly with each other. The database (or a shared blackboard/repository) is the primary means of integration — components don't know about each other, only about the shared state.

## Key Concepts

- **Central Repository:** The shared data store every component reads from and writes to
- **Blackboard:** A specific data-centric style where independent "knowledge sources" post partial results to a shared space until a solution emerges
- **Independent Knowledge Source:** A component that watches the shared store and contributes when it recognizes something it can act on
- **Data as Integration:** Components integrate by sharing a schema, not by calling each other's APIs
- **Schema-on-Write vs. Schema-on-Read:** Whether structure is enforced when data is stored or interpreted later when it's read
- **Shared State:** All coordination flows through the data store, which becomes the system's single source of truth and its single biggest coupling point

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                       Blackboard (shared state)               │
└───────┬─────────────────┬─────────────────┬──────────────────┘
        ▲  reads/writes    ▲  reads/writes    ▲  reads/writes
        │                  │                  │
 ┌──────┴─────┐     ┌──────┴─────┐     ┌──────┴─────┐
 │ Knowledge   │     │ Knowledge   │     │ Knowledge   │
 │ Source A    │     │ Source B    │     │ Source C    │
 │ (pricing)   │     │ (inventory) │     │ (fraud check)│
 └────────────┘     └────────────┘     └────────────┘
   components never call each other directly — only the shared store
```

## Example: Blackboard-Style Coordination

```typescript
// Independent components ("knowledge sources") never call each other —
// they only observe and update the shared blackboard.
interface OrderRecord {
  id: string;
  items: Array<{ sku: string; quantity: number; unitPrice: number }>;
  total?: number;
  fraudChecked?: boolean;
  inventoryConfirmed?: boolean;
  status: 'draft' | 'priced' | 'ready' | 'approved';
}

class Blackboard {
  private records = new Map<string, OrderRecord>();
  private watchers: Array<(record: OrderRecord) => void> = [];

  put(record: OrderRecord): void {
    this.records.set(record.id, record);
    this.watchers.forEach(watch => watch(record));
  }

  get(id: string): OrderRecord | undefined {
    return this.records.get(id);
  }

  onUpdate(watcher: (record: OrderRecord) => void): void {
    this.watchers.push(watcher);
  }
}

// Knowledge source: prices the order once items are present
class PricingKnowledgeSource {
  constructor(private board: Blackboard) {
    board.onUpdate(record => {
      if (record.status === 'draft' && record.total === undefined) {
        const total = record.items.reduce((sum, i) => sum + i.unitPrice * i.quantity, 0);
        board.put({ ...record, total, status: 'priced' });
      }
    });
  }
}

// Knowledge source: confirms inventory once the order is priced
class InventoryKnowledgeSource {
  constructor(private board: Blackboard) {
    board.onUpdate(record => {
      if (record.status === 'priced' && !record.inventoryConfirmed) {
        board.put({ ...record, inventoryConfirmed: true, status: 'ready' });
      }
    });
  }
}

// Usage
const board = new Blackboard();
new PricingKnowledgeSource(board);
new InventoryKnowledgeSource(board);

board.put({
  id: 'order-1',
  items: [{ sku: 'sku-1', quantity: 2, unitPrice: 25 }],
  status: 'draft'
});
// PricingKnowledgeSource reacts -> status 'priced' -> InventoryKnowledgeSource reacts -> status 'ready'
```

## Example: Shared-Database Integration

```typescript
// Multiple independent services integrate purely through a shared table —
// contrast with Microservices' database-per-service isolation.
class OrderWriteService {
  constructor(private db: Database) {}

  async createOrder(order: OrderRecord): Promise<void> {
    await this.db.query(
      'INSERT INTO orders (id, items, status) VALUES ($1, $2, $3)',
      [order.id, JSON.stringify(order.items), 'draft']
    );
  }
}

// A completely separate service, deployed independently, reads the same table directly —
// no API call between them, the schema itself is the contract.
class ReportingService {
  constructor(private db: Database) {}

  async dailyOrderTotals(date: string): Promise<{ status: string; count: number }[]> {
    return this.db.query(
      `SELECT status, COUNT(*) as count FROM orders
       WHERE created_at::date = $1 GROUP BY status`,
      [date]
    );
  }
}
```

## Advantages

- **Simple Integration Model:** New components just read/write the shared schema — no API contracts to design
- **Single Source of Truth:** No synchronization problem between multiple copies of the same data
- **Good Fit for Exploratory Problems:** The blackboard style works well when the solution path isn't known upfront (AI planning, speech recognition, complex diagnosis)
- **Easy Reporting/Analytics:** Everything lives in one place, making cross-cutting queries straightforward

## Disadvantages

- **Tight Schema Coupling:** Every component depends on the same schema; changing it risks breaking components you don't own
- **No Independent Deployment:** A schema migration can require coordinating every consuming component at once
- **Scaling Bottleneck:** The central store becomes the throughput ceiling for the entire system
- **Hidden Dependencies:** Because components never call each other directly, the actual dependency graph is invisible in code — it only exists in the schema

## When to Use

- Systems where multiple independent processes need to collaborate on a partial, evolving solution (planning systems, diagnostic engines)
- Reporting/analytics platforms where many consumers need direct, flexible access to the same data
- Small systems where an API layer between components would add cost without real benefit
- Legacy integration where existing components already only know how to talk to a shared database

## Best Practices

- **Version the Schema Deliberately:** Treat schema changes with the same care as breaking API changes
- **Document Implicit Dependencies:** Since the code doesn't show who reads/writes what, maintain an explicit map of components to tables/fields
- **Guard Write Access:** Not every component should be allowed to write every field — enforce boundaries even within a shared store
- **Prefer Read-Only Sharing Where Possible:** Give reporting/analytics consumers read replicas instead of direct write access to operational tables

## Common Mistakes

- **Uncontrolled Shared Writes:** Multiple components writing to the same fields with no ownership, producing inconsistent state
- **Schema as an Afterthought:** Treating the shared schema as an implementation detail instead of the actual integration contract it is
- **Silent Coupling:** Assuming components are "decoupled" because they don't call each other directly, while they're tightly bound through the schema
- **Scaling by Adding More Readers:** Ignoring that a shared central store doesn't scale the same way independent, owned data stores do

## Related Patterns

- [Repository Pattern](../../patterns/repository-pattern/repository-pattern.md) — a data-access abstraction commonly layered on top of a shared store
- [Microservices Architecture](../microservices-architecture/microservices-architecture.md) — the opposite extreme: database-per-service instead of one shared store
- [Event Sourcing Pattern](../../patterns/event-sourcing-pattern/event-sourcing-pattern.md) — an alternative way to make state changes the shared source of truth without a shared mutable schema

## References & Sources

- Frank Buschmann et al. — "Pattern-Oriented Software Architecture, Volume 1" (1996), Blackboard Pattern
- Mark Richards & Neal Ford — "Fundamentals of Software Architecture" (2020), Data-Centric chapter
- Zhamak Dehghani — "Data Mesh" (2022), a modern counter-movement away from centralized data stores
