# Event-Driven Architecture (EDA)

An event-driven architecture uses events to trigger and communicate between decoupled services. An event is a change in state, like an item being placed in a shopping cart on an e-commerce website.

## Key Concepts

- **Event:** Notification of a state change (OrderCreated, PaymentProcessed, etc.)
- **Event Producer:** Component that emits events
- **Event Consumer:** Component that listens to and handles events
- **Event Broker:** Mediates between producers and consumers (message queue, event bus)
- **Decoupling:** Services don't know about each other, only about events

## Architecture Patterns

### 1. Event Broker Pattern

Broker mediates communication between event producers and consumers.

```
Producer → Event Broker → Consumer
                       → Consumer
```

**Components:**
- Producers publish events to broker
- Broker routes events to subscribed consumers
- Consumers handle events asynchronously

**Tools:** RabbitMQ, Apache Kafka, AWS SQS

### 2. Event Sourcing

Store all state changes as immutable sequence of events instead of storing current state.

```typescript
// Instead of storing: user { id: 1, balance: 1000 }
// Store events:
events = [
  { type: 'UserCreated', userId: 1, initialBalance: 1000 },
  { type: 'DepositMade', userId: 1, amount: 500 },
  { type: 'WithdrawalMade', userId: 1, amount: 200 }
];

// Reconstruct state by replaying events
let balance = 0;
events.forEach(e => {
  if (e.type === 'UserCreated') balance = e.initialBalance;
  if (e.type === 'DepositMade') balance += e.amount;
  if (e.type === 'WithdrawalMade') balance -= e.amount;
});
```

## Example Flow

```
E-Commerce Order Flow:
┌─────────────┐
│   Order     │
│  Service    │
└──────┬──────┘
       │ publishes: OrderCreated
       ▼
   Event Broker
       │
   ┌───┴────────┬──────────────┐
   │            │              │
   ▼            ▼              ▼
Payment    Inventory     Notification
Service    Service       Service
   │            │              │
   ├────────────┴──────────────┤
   │
   ▼
   All services updated independently
```

## Advantages

- **Loose Coupling:** Services are independent
- **Scalability:** Easy to add new consumers
- **Resilience:** One service failure doesn't block others
- **Audit Trail:** Event log provides history
- **Real-time Processing:** Immediate reactions to events

## Disadvantages

- **Eventual Consistency:** Data not immediately consistent across services
- **Complexity:** Harder to debug distributed flows
- **Testing:** Difficult to test across async boundaries
- **Message Ordering:** Hard to maintain event order across services

## Common Use Cases

- **E-commerce:** Order → Payment → Inventory → Shipping
- **User Activities:** Sign-up → Email verification → Welcome email
- **Real-time Analytics:** User events → Analytics pipeline
- **Monitoring:** System events → Alerts → Notifications
- **IoT:** Sensor events → Processing → Alerts

## Implementation Tools

- **Message Brokers:** RabbitMQ, Apache Kafka, AWS SNS/SQS
- **Event Streaming:** Kafka, AWS Kinesis
- **In-app Event Bus:** Node EventEmitter, RxJS Subjects
- **Cloud:** AWS EventBridge, Google Pub/Sub, Azure Event Grid
