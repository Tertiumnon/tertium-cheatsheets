# Publish-Subscribe (Pub/Sub) Pattern

Publish-Subscribe decouples publishers and subscribers through an intermediary broker/topic, so neither side ever holds a reference to the other. It's easy to confuse with the [Observer Pattern](../observer-pattern/observer-pattern.md), but the distinction matters: in Observer, the subject holds direct references to its observers and calls them synchronously; in Pub/Sub, publishers only know a topic name, subscribers only know a topic name, and a broker in between handles delivery — often asynchronously, often across process or network boundaries.

## Problem

- Publishers and consumers need to communicate without knowing about each other
- The number and identity of consumers can change at runtime, or grow beyond one
- Communication may need to cross process, service, or network boundaries
- Direct references (as in Observer) don't work when publisher and subscriber live in different processes

## Solution

Introduce a broker that owns named topics/channels. Publishers send messages to a topic without knowing who (if anyone) is subscribed. Subscribers register interest in a topic without knowing who publishes to it.

## Structure

```
Publisher A ──┐                              ┌──▶ Subscriber 1
Publisher B ──┼──▶ [ Topic: "order.created" ]─┼──▶ Subscriber 2
Publisher C ──┘         (broker)              └──▶ Subscriber 3

Neither side holds a reference to the other — only to the topic name.
```

## Observer vs. Pub/Sub

| Aspect | Observer | Pub/Sub |
|---|---|---|
| Coupling | Subject holds direct references to observers | Publisher/subscriber only know a topic name |
| Location | Same process, typically | Same process or across processes/services |
| Delivery | Usually synchronous | Often asynchronous, via a broker |
| Cardinality | One subject to many observers | Many publishers to many subscribers, mediated by topics |
| Failure isolation | An observer throwing can affect the subject's call stack | Broker typically isolates subscriber failures from publishers |

## Basic Implementation: In-Memory Topic Broker

```typescript
type Handler<T> = (payload: T) => void;

class TopicBroker {
  private topics = new Map<string, Set<Handler<unknown>>>();

  subscribe<T>(topic: string, handler: Handler<T>): () => void {
    if (!this.topics.has(topic)) this.topics.set(topic, new Set());
    this.topics.get(topic)!.add(handler as Handler<unknown>);

    return () => this.topics.get(topic)?.delete(handler as Handler<unknown>);
  }

  publish<T>(topic: string, payload: T): void {
    // Publisher has no idea how many subscribers exist, or whether there are any at all
    this.topics.get(topic)?.forEach(handler => handler(payload));
  }
}

// Usage
interface OrderCreated { orderId: string; total: number }

const broker = new TopicBroker();

const unsubscribeEmail = broker.subscribe<OrderCreated>('order.created', (event) => {
  console.log(`Sending confirmation email for order ${event.orderId}`);
});

broker.subscribe<OrderCreated>('order.created', (event) => {
  console.log(`Reserving inventory for order ${event.orderId}`);
});

// Publisher only knows the topic name, not who (if anyone) is listening
broker.publish('order.created', { orderId: 'order-1', total: 129.99 });

unsubscribeEmail();
```

## Real-World: Message Broker Client (Kafka/RabbitMQ/SNS-style)

```typescript
// A thin abstraction over a real broker — the application code never talks to
// Kafka/RabbitMQ/SNS directly, only to this interface
interface MessageBroker {
  publish(topic: string, message: Record<string, unknown>): Promise<void>;
  subscribe(topic: string, handler: (message: Record<string, unknown>) => Promise<void>): void;
}

class OrderService {
  constructor(private broker: MessageBroker) {}

  async createOrder(customerId: string, items: unknown[]): Promise<string> {
    const orderId = crypto.randomUUID();
    // ... persist the order ...
    await this.broker.publish('order.created', { orderId, customerId, items });
    return orderId;
  }
}

class InventoryService {
  constructor(broker: MessageBroker) {
    // Subscribes without ever knowing which service(s) publish to this topic
    broker.subscribe('order.created', async (message) => {
      console.log(`Reserving inventory for order ${message.orderId}`);
    });
  }
}

class NotificationService {
  constructor(broker: MessageBroker) {
    broker.subscribe('order.created', async (message) => {
      console.log(`Notifying customer ${message.customerId}`);
    });
  }
}
```

## Advantages

- **Full Decoupling:** Publishers and subscribers never reference each other, only a topic name
- **Dynamic Fan-Out:** New subscribers can be added without changing the publisher at all
- **Location Transparency:** Works the same whether publisher and subscriber are in the same process or across the network
- **Resilience:** A broker can buffer messages so a temporarily-down subscriber doesn't lose them (depending on broker guarantees)

## Disadvantages

- **Delivery Guarantees Vary:** At-most-once, at-least-once, and exactly-once semantics differ by broker and must be understood explicitly
- **Harder to Trace:** Following a message from publisher to every subscriber requires broker-level tooling, not just reading code
- **Ordering Isn't Guaranteed by Default:** Multiple publishers/partitions can reorder messages unless the broker and topic design account for it
- **Operational Dependency:** Introduces a broker as new infrastructure to run, scale, and monitor

## When to Use

- Multiple independent consumers need to react to the same event without the publisher knowing who they are
- Publisher and subscriber live in different processes, services, or machines
- The set of subscribers changes over time or isn't known at publish time
- Building the backbone of an [Event-Driven Architecture](../../arhitectures/event-driven-architecture/event-driven-architecture.md)

## Best Practices

- **Name Topics by Fact, Not Command:** `order.created`, not `notify-customer` — topics describe what happened, not who should act
- **Design for At-Least-Once Delivery:** Make subscriber handlers idempotent since most brokers can redeliver
- **Keep Payloads Versioned:** Treat the message schema like an API contract
- **Isolate Subscriber Failures:** One failing subscriber shouldn't block delivery to others
- **Monitor Consumer Lag:** Know how far behind each subscriber is from the latest published message

## Common Mistakes

- **Treating Pub/Sub Like a Direct Call:** Assuming a publish will be handled synchronously or exactly once with no extra handling
- **God Topics:** Publishing every kind of event to one giant topic instead of meaningful, narrowly-scoped topics
- **No Idempotency:** Subscriber logic breaks the second time it processes the same redelivered message
- **Confusing with Observer:** Reaching for a broker when a simple in-process Observer would do, adding operational overhead for no benefit

## Related Patterns

- [Observer Pattern](../observer-pattern/observer-pattern.md) — the direct-reference, same-process cousin of Pub/Sub
- [Producer-Consumer Pattern](../producer-consumer-pattern/producer-consumer.md) — related but focused on work distribution rather than broadcast notification
- [Event-Driven Architecture](../../arhitectures/event-driven-architecture/event-driven-architecture.md) — the architectural style Pub/Sub typically implements
- [Event Sourcing Pattern](../event-sourcing-pattern/event-sourcing-pattern.md) — often uses Pub/Sub to propagate events to projections

## References & Sources

- Gregor Hohpe & Bobby Woolf — "Enterprise Integration Patterns" (2003), Publish-Subscribe Channel
- Google Cloud — Pub/Sub Documentation: https://cloud.google.com/pubsub/docs/overview
- AWS — Simple Notification Service (SNS): https://docs.aws.amazon.com/sns/
- Apache Kafka Documentation: https://kafka.apache.org/documentation/
