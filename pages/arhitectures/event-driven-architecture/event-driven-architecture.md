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

## Folder/File Structure

Organize event-driven systems by services with clear event definitions and handlers:

```
src/
├── services/                         # Microservices
│   ├── order-service/                # Order Service
│   │   ├── events/                   # Event definitions
│   │   │   ├── order-created.event.ts
│   │   │   ├── order-confirmed.event.ts
│   │   │   └── order-cancelled.event.ts
│   │   ├── handlers/                 # Event handlers
│   │   │   ├── handle-inventory-updated.handler.ts
│   │   │   └── handle-payment-processed.handler.ts
│   │   ├── publishers/               # Event publishers
│   │   │   └── order.publisher.ts
│   │   ├── controllers/              # API endpoints
│   │   │   └── order.controller.ts
│   │   ├── services/                 # Business logic
│   │   │   └── order.service.ts
│   │   └── models/                   # Data models
│   │       └── order.model.ts
│   ├── payment-service/              # Payment Service
│   │   ├── events/
│   │   │   ├── payment-processed.event.ts
│   │   │   └── payment-failed.event.ts
│   │   ├── handlers/                 # Listen to events
│   │   │   └── handle-order-created.handler.ts
│   │   ├── publishers/
│   │   │   └── payment.publisher.ts
│   │   ├── controllers/
│   │   │   └── payment.controller.ts
│   │   └── services/
│   │       └── payment.service.ts
│   ├── inventory-service/            # Inventory Service
│   │   ├── events/
│   │   │   ├── inventory-updated.event.ts
│   │   │   └── stock-low.event.ts
│   │   ├── handlers/
│   │   │   └── handle-order-created.handler.ts
│   │   ├── publishers/
│   │   │   └── inventory.publisher.ts
│   │   ├── controllers/
│   │   │   └── inventory.controller.ts
│   │   └── services/
│   │       └── inventory.service.ts
│   └── notification-service/         # Notification Service
│       ├── handlers/                 # Consume events
│       │   ├── handle-order-created.handler.ts
│       │   └── handle-payment-processed.handler.ts
│       ├── services/
│       │   └── email.service.ts
│       └── templates/
│           └── order-confirmation.template.html
├── shared/                           # Shared across services
│   ├── events/                       # Event definitions & types
│   │   ├── base.event.ts             # Abstract event class
│   │   ├── domain-events.ts          # All domain events
│   │   └── event.interface.ts        # Event contract
│   ├── event-bus/                    # Event broker implementation
│   │   ├── event-bus.interface.ts
│   │   ├── event-bus.implementation.ts
│   │   ├── rabbitmq.bus.ts
│   │   ├── kafka.bus.ts
│   │   └── in-memory.bus.ts          # For testing
│   ├── event-handler/                # Handler registration
│   │   ├── event-handler.registry.ts
│   │   └── event-handler.decorator.ts
│   ├── event-store/                  # Event sourcing
│   │   ├── event-store.interface.ts
│   │   ├── event-store.implementation.ts
│   │   └── event-repository.ts
│   └── types/                        # Shared types
│       └── common.types.ts
└── config/                           # Configuration
    ├── event-bus.config.ts
    └── services.config.ts
```

### Event Definition Example

```
shared/events/
├── order-created.event.ts
├── payment-processed.event.ts
├── inventory-updated.event.ts
└── notification-sent.event.ts

// shared/events/base.event.ts
export abstract class DomainEvent {
  public readonly occurredAt: Date = new Date();
  constructor(
    public readonly aggregateId: string,
    public readonly eventType: string
  ) {}
}

// shared/events/order-created.event.ts
export class OrderCreatedEvent extends DomainEvent {
  constructor(
    aggregateId: string,
    public readonly customerId: string,
    public readonly amount: number
  ) {
    super(aggregateId, 'OrderCreated');
  }
}
```

### Service File Organization

Each service follows this pattern:
- **events/** - Events this service publishes
- **handlers/** - Event handlers (subscribes to other services' events)
- **publishers/** - Logic to publish events to the event bus
- **controllers/** - HTTP endpoints
- **services/** - Business logic
- **models/** - Data models and schemas

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

## References & Sources

### Books & Publications
- **Gregor Hohpe & Bobby Woolf** - "Enterprise Integration Patterns" (2003) - Foundational event-driven patterns
- **Sam Newman** - "Building Microservices: Designing Fine-Grained Systems" (2015) - EDA in distributed systems
- **Martin Fowler** - "Event Sourcing" - https://martinfowler.com/eaaDev/EventSourcing.html

### Articles & Official Documentation
- Martin Fowler - Event-Driven Architecture: https://martinfowler.com/articles/201701-event-driven.html
- AWS - Event-Driven Architecture: https://aws.amazon.com/event-driven-architecture/
- Apache Kafka Documentation: https://kafka.apache.org/documentation/
- RabbitMQ Documentation: https://www.rabbitmq.com/documentation.html
- Microsoft - Event-Driven Architecture: https://docs.microsoft.com/en-us/azure/architecture/guide/architecture-styles/event-driven

### Related Patterns
- Event Sourcing
- CQRS (Command Query Responsibility Segregation)
- Saga Pattern (Distributed Transactions)
- Choreography vs. Orchestration
- Message Broker Patterns
- Domain Events
