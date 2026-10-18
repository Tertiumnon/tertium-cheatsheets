# Event Sourcing Pattern

Event Sourcing stores every change to application state as a sequence of immutable events instead of storing just the current state. Current state is derived by replaying those events from the beginning (or from a snapshot), which gives a complete audit trail for free and lets the same event stream feed multiple read models — commonly paired with [CQRS](../cqrs-pattern/cqrs-pattern.md).

## Problem

- Storing only current state discards the history of how it got there — useful for audits, debugging, and undo
- Different parts of the system need different projections of the same history (a balance, a statement, a fraud-detection score)
- Retrofitting an audit log onto a current-state model after the fact is incomplete and easy to get wrong
- Concurrent updates to current state are hard to reconcile without knowing what actually happened, in order

## Solution

Persist every state transition as an immutable, ordered event. Rebuild current state by replaying events through a fold function; any number of independent projections can consume the same stream to build their own views.

## Structure

```
Command ──▶ Aggregate ──▶ Event(s) ──▶ Event Store (append-only)
                                            │
                              ┌─────────────┼─────────────┐
                              ▼             ▼             ▼
                        Replay to      Projection A   Projection B
                        rebuild         (read model)   (analytics)
                        aggregate state
```

## Basic Implementation: Event Store and Replay

```typescript
// Events are facts that already happened — named in the past tense, never mutated
interface DomainEvent {
  aggregateId: string;
  type: string;
  payload: Record<string, unknown>;
  occurredAt: Date;
  version: number;
}

class EventStore {
  private events: DomainEvent[] = [];

  append(event: DomainEvent): void {
    this.events.push(event);
  }

  readStream(aggregateId: string): DomainEvent[] {
    return this.events
      .filter(e => e.aggregateId === aggregateId)
      .sort((a, b) => a.version - b.version);
  }
}

// Aggregate: rebuilds its state purely by folding over its own event stream
class BankAccount {
  private balance = 0;
  private version = 0;
  private pendingEvents: DomainEvent[] = [];

  static rehydrate(accountId: string, history: DomainEvent[]): BankAccount {
    const account = new BankAccount(accountId);
    history.forEach(event => account.apply(event, false));
    return account;
  }

  constructor(private accountId: string) {}

  open(initialDeposit: number): void {
    this.raise('AccountOpened', { initialDeposit });
  }

  deposit(amount: number): void {
    if (amount <= 0) throw new Error('Deposit must be positive');
    this.raise('MoneyDeposited', { amount });
  }

  withdraw(amount: number): void {
    if (amount > this.balance) throw new Error('Insufficient funds');
    this.raise('MoneyWithdrawn', { amount });
  }

  getBalance(): number { return this.balance; }
  getUncommittedEvents(): DomainEvent[] { return this.pendingEvents; }

  private raise(type: string, payload: Record<string, unknown>): void {
    const event: DomainEvent = {
      aggregateId: this.accountId,
      type,
      payload,
      occurredAt: new Date(),
      version: this.version + 1
    };
    this.apply(event, true);
  }

  private apply(event: DomainEvent, isNew: boolean): void {
    switch (event.type) {
      case 'AccountOpened':
        this.balance = event.payload.initialDeposit as number;
        break;
      case 'MoneyDeposited':
        this.balance += event.payload.amount as number;
        break;
      case 'MoneyWithdrawn':
        this.balance -= event.payload.amount as number;
        break;
    }
    this.version = event.version;
    if (isNew) this.pendingEvents.push(event);
  }
}

// Usage
const store = new EventStore();

const account = new BankAccount('acc-1');
account.open(100);
account.deposit(50);
account.withdraw(30);
account.getUncommittedEvents().forEach(e => store.append(e));

// Later, or on another node: rebuild state purely from history
const history = store.readStream('acc-1');
const rehydrated = BankAccount.rehydrate('acc-1', history);
console.log(rehydrated.getBalance()); // 120
```

## Snapshotting for Performance

```typescript
// Replaying thousands of events on every load is expensive — snapshot periodically
interface Snapshot {
  aggregateId: string;
  version: number;
  state: { balance: number };
}

class SnapshotStore {
  private snapshots = new Map<string, Snapshot>();

  save(snapshot: Snapshot): void {
    this.snapshots.set(snapshot.aggregateId, snapshot);
  }

  load(aggregateId: string): Snapshot | undefined {
    return this.snapshots.get(aggregateId);
  }
}

function rehydrateWithSnapshot(
  accountId: string,
  eventStore: EventStore,
  snapshotStore: SnapshotStore
): BankAccount {
  const snapshot = snapshotStore.load(accountId);
  const allEvents = eventStore.readStream(accountId);

  // Only replay events that happened after the snapshot was taken, instead of from the beginning.
  // A production version would also seed BankAccount's starting balance/version from
  // snapshot.state before folding the remaining events; omitted here for brevity.
  const eventsSinceSnapshot = snapshot
    ? allEvents.filter(e => e.version > snapshot.version)
    : allEvents;

  return BankAccount.rehydrate(accountId, eventsSinceSnapshot);
}
```

## Advantages

- **Complete Audit Trail:** Every change is recorded, in order, forever — nothing is silently overwritten
- **Temporal Queries:** State can be reconstructed as of any point in time by replaying up to that event
- **Multiple Projections:** The same event stream can feed any number of independent read models
- **Natural Fit with CQRS:** Events are exactly the mechanism CQRS needs to keep read models in sync

## Disadvantages

- **Query Complexity:** Getting current state requires replay (or a maintained projection) instead of a simple row lookup
- **Event Schema Evolution:** Old events must remain readable forever as the domain model evolves — versioning is a real, ongoing cost
- **Storage Growth:** An append-only log grows indefinitely; snapshotting and archival strategy become necessary
- **Steep Learning Curve:** Thinking in events instead of current state is a genuine mental shift for most teams

## When to Use

- Domains where audit history is a business requirement, not just a nice-to-have (finance, healthcare, compliance)
- Systems that need to answer "what was the state at time X" or "why did this happen"
- Already using [CQRS](../cqrs-pattern/cqrs-pattern.md) and needing a reliable way to keep multiple read models in sync
- Complex domains where debugging "how did we get into this state" is a recurring, expensive problem

## Best Practices

- **Name Events in the Past Tense:** `OrderShipped`, not `ShipOrder` — events are facts, not commands
- **Never Mutate or Delete Events:** Corrections are new compensating events, not edits to history
- **Version Your Events:** Plan for schema evolution from day one (upcasting old event shapes to new ones)
- **Snapshot Long Streams:** Avoid replaying thousands of events on every load
- **Keep Events Small and Focused:** One event, one fact — don't bundle unrelated state changes together

## Common Mistakes

- **Treating Events as CRUD Logs:** Recording generic `EntityUpdated` events with a full-state diff instead of meaningful domain facts
- **Mutable Event Store:** Allowing events to be edited or deleted, defeating the audit-trail guarantee
- **No Snapshot Strategy:** Letting hot aggregates accumulate years of events with no way to load them quickly
- **Coupling Consumers to Internal Event Shape:** Not versioning events, breaking every consumer the moment the schema changes

## Related Patterns

- [CQRS Pattern](../cqrs-pattern/cqrs-pattern.md) — the read side that typically consumes an event-sourced write side
- [Event-Driven Architecture](../../arhitectures/event-driven-architecture/event-driven-architecture.md) — the broader style event sourcing operates within
- [Domain-Driven Architecture](../../arhitectures/domain-driven-architecture/domain-driven-architecture.md) — aggregates are the natural boundary for an event stream

## References & Sources

- Martin Fowler — Event Sourcing: https://martinfowler.com/eaaDev/EventSourcing.html
- Greg Young — CQRS and Event Sourcing (talks and documents): https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf
- EventStoreDB Documentation: https://developers.eventstore.com/
