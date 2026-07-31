# Distributed Systems Architecture

Distributed Systems Architecture spans any system split across multiple networked computers that coordinate to appear as one coherent system from the outside. Microservices, SOA, and distributed databases are all concrete strategies within this broader style — this page covers the problems any of them must solve regardless of the specific style chosen: network unreliability, partial failure, consistency, and coordination between independent nodes.

## Key Concepts

- **Node:** An independent computer/process participating in the system
- **Network Partition:** A break in communication between nodes that are both still running
- **CAP Theorem:** Under a network partition, a system can guarantee either Consistency or Availability, not both
- **Consistency Models:** Strong consistency (all nodes see the same data immediately) vs. eventual consistency (nodes converge over time)
- **Replication:** Copying data across nodes for availability and fault tolerance
- **Consensus:** Algorithms (Raft, Paxos) that let a set of nodes agree on a single value despite failures
- **Partial Failure:** Some nodes fail while others keep running — the defining problem distributed systems must handle that single-process systems don't
- **Idempotency:** An operation that produces the same result no matter how many times a client retries it after a timeout

## Architecture Diagram

```
   ┌────────┐        ┌────────┐        ┌────────┐
   │ Node A │◀──────▶│ Node B │◀──────▶│ Node C │
   └────────┘        └───┬────┘        └────────┘
                          │
                     ╳ network partition ╳
                          │
                     ┌────▼───┐
                     │ Node D │   ← isolated, but still running
                     └────────┘
```

Every node can independently fail, be slow, or become unreachable — the architecture has to assume this will happen, not treat it as exceptional.

## The Fallacies of Distributed Computing

Assumptions that inevitably break and that distributed architecture must design around:

1. The network is reliable
2. Latency is zero
3. Bandwidth is infinite
4. The network is secure
5. Topology doesn't change
6. There is one administrator
7. Transport cost is zero
8. The network is homogeneous

## Idempotent Request Handling

```typescript
// A client retrying after a timeout must not cause the operation to run twice
interface PaymentRequest {
  idempotencyKey: string;
  amount: number;
  accountId: string;
}

class PaymentProcessor {
  private processedKeys = new Map<string, PaymentResult>();

  async charge(request: PaymentRequest): Promise<PaymentResult> {
    const cached = this.processedKeys.get(request.idempotencyKey);
    if (cached) {
      // Same request arriving again (client retried after a dropped response) — return the original result
      return cached;
    }

    const result = await this.executeCharge(request);
    this.processedKeys.set(request.idempotencyKey, result);
    return result;
  }

  private async executeCharge(request: PaymentRequest): Promise<PaymentResult> {
    return { success: true, transactionId: crypto.randomUUID(), amount: request.amount };
  }
}
```

## Retry with Timeout and Backoff

```typescript
// Assumes calls can hang or fail transiently — never assume the network just works
async function callWithTimeout<T>(
  fn: () => Promise<T>,
  timeoutMs: number,
  retries = 3
): Promise<T> {
  for (let attempt = 1; attempt <= retries; attempt++) {
    try {
      return await Promise.race([
        fn(),
        new Promise<never>((_, reject) =>
          setTimeout(() => reject(new Error('Request timed out')), timeoutMs)
        )
      ]);
    } catch (error) {
      if (attempt === retries) throw error;
      const delay = 100 * 2 ** attempt;
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
  throw new Error('Unreachable');
}
```

## Quorum Read/Write

```typescript
// Simplified quorum: write succeeds once W of N replicas ack, read succeeds once R of N respond
// Choosing W + R > N guarantees a read overlaps with the most recent write
class ReplicatedStore {
  constructor(
    private replicas: DataReplica[],
    private writeQuorum: number,
    private readQuorum: number
  ) {}

  async write(key: string, value: string): Promise<void> {
    const acks = await Promise.allSettled(this.replicas.map(r => r.write(key, value)));
    const successCount = acks.filter(a => a.status === 'fulfilled').length;
    if (successCount < this.writeQuorum) {
      throw new Error('Write quorum not reached');
    }
  }

  async read(key: string): Promise<string | null> {
    const results = await Promise.allSettled(this.replicas.map(r => r.read(key)));
    const values = results
      .filter((r): r is PromiseFulfilledResult<string | null> => r.status === 'fulfilled')
      .map(r => r.value);

    if (values.length < this.readQuorum) {
      throw new Error('Read quorum not reached');
    }
    // Return the most recently written value among the quorum (last-write-wins, simplified)
    return values[values.length - 1];
  }
}

interface DataReplica {
  write(key: string, value: string): Promise<void>;
  read(key: string): Promise<string | null>;
}
```

## Advantages

- **Scalability:** Load and data are spread across many machines instead of one
- **Fault Tolerance:** Replication means the system can survive individual node failures
- **Geographic Distribution:** Nodes can be placed close to users for lower latency
- **Elastic Capacity:** Nodes can be added or removed based on demand

## Disadvantages

- **Partial Failure Complexity:** Every call must account for the possibility of the remote side being down, slow, or unreachable
- **Consistency Tradeoffs:** CAP theorem forces an explicit choice — you can't have strong consistency and full availability during a partition
- **Debugging Difficulty:** Failures and race conditions span multiple machines and clocks, making root-causing much harder
- **Operational Overhead:** Requires monitoring, tracing, and coordination infrastructure a single-process system doesn't need

## When to Use

- Systems that must survive individual machine or data-center failures
- Workloads too large for a single machine's compute or storage capacity
- Global user bases needing low-latency access from multiple regions
- Any system already committing to [Microservices](../microservices-architecture/microservices-architecture.md) or [Service-Oriented Architecture](../service-oriented-architecture/service-oriented-architecture.md) — those styles are distributed systems by definition

## Best Practices

- **Design for Partial Failure:** Every remote call needs a timeout, retry policy, and fallback
- **Prefer Idempotent Operations:** Makes retries safe by default
- **Be Explicit About Consistency:** Document whether a given read path is strongly or eventually consistent
- **Use Correlation IDs:** Trace a single logical request across every node it touches
- **Test Failure, Not Just Success:** Chaos-test network partitions and node crashes before production does it for you

## Common Mistakes

- **Assuming the Network Is Reliable:** Treating remote calls like local function calls with no failure handling
- **Ignoring Clock Skew:** Relying on wall-clock timestamps across nodes to determine event order
- **No Idempotency Keys:** Retrying non-idempotent operations and double-charging, double-shipping, or double-emailing
- **Synchronous Chains:** Building long synchronous call chains across nodes, multiplying latency and failure surface

## Related Patterns

- [Microservices Architecture](../microservices-architecture/microservices-architecture.md) — a concrete distributed-systems strategy
- [Service-Oriented Architecture](../service-oriented-architecture/service-oriented-architecture.md) — an earlier distributed-systems strategy
- [Event-Driven Architecture](../event-driven-architecture/event-driven-architecture.md) — a common communication style between distributed nodes
- [CQRS Pattern](../../patterns/cqrs-pattern/cqrs-pattern.md) and [Event Sourcing Pattern](../../patterns/event-sourcing-pattern/event-sourcing-pattern.md) — consistency-management patterns commonly used in distributed systems

## References & Sources

- Martin Kleppmann — "Designing Data-Intensive Applications" (2017)
- Eric Brewer — CAP Theorem: https://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed/
- Leslie Lamport — "Time, Clocks, and the Ordering of Events in a Distributed System" (1978)
- Arnon Rotem-Gal-Oz — "Fallacies of Distributed Computing Explained": https://www.rgoarchitects.com/Files/fallacies.pdf
