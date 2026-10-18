# Orchestration vs. Choreography Pattern

Orchestration and Choreography are two opposing strategies for coordinating a business process that spans multiple services — most commonly a Saga, a sequence of local transactions with compensating actions if a later step fails. Orchestration uses a central coordinator that tells each service what to do and when. Choreography has no central authority: each service reacts to events published by others, and coordination emerges from those independent reactions.

## Problem

- A business process (place order → charge payment → reserve inventory → ship) spans multiple services, each owning its own data
- No distributed transaction/two-phase commit is available (or desirable) across services
- If a later step fails, earlier steps need to be compensated (refunded, released, cancelled)
- Someone needs to own the sequencing and failure handling for the overall process

## Solution

Choose one of two coordination strategies:

- **Orchestration:** A dedicated orchestrator calls each service in sequence, tracks progress, and triggers compensations on failure. The process logic lives in one place.
- **Choreography:** Each service publishes an event when it finishes its part; other services subscribe to the events they care about and react independently. No single place holds the full process logic.

## Structure

```
Orchestration                              Choreography
┌───────────────┐                          ┌────────┐  OrderCreated   ┌───────────┐
│  Orchestrator  │──calls──▶ Payment        │ Order  │───────────────▶│ Payment    │
│  (Saga owner)  │──calls──▶ Inventory      └────────┘                 └─────┬─────┘
│                │──calls──▶ Shipping                                PaymentProcessed
│  owns sequence │◀─results──                                              │
│  & compensation│                                                          ▼
└───────────────┘                                                    ┌───────────┐
                                                                       │ Inventory  │
                                                                       └─────┬─────┘
                                                                    InventoryReserved
                                                                             │
                                                                             ▼
                                                                       ┌───────────┐
                                                                       │ Shipping   │
                                                                       └───────────┘
                                             no single service knows the whole process
```

## Orchestration Implementation

```typescript
// One class owns the entire process, including how to undo each completed step
interface SagaStep {
  name: string;
  execute(context: OrderContext): Promise<void>;
  compensate(context: OrderContext): Promise<void>;
}

interface OrderContext {
  orderId: string;
  customerId: string;
  amount: number;
}

class OrderSagaOrchestrator {
  private steps: SagaStep[] = [
    {
      name: 'chargePayment',
      execute: async (ctx) => paymentService.charge(ctx.customerId, ctx.amount),
      compensate: async (ctx) => paymentService.refund(ctx.customerId, ctx.amount)
    },
    {
      name: 'reserveInventory',
      execute: async (ctx) => inventoryService.reserve(ctx.orderId),
      compensate: async (ctx) => inventoryService.release(ctx.orderId)
    },
    {
      name: 'scheduleShipping',
      execute: async (ctx) => shippingService.schedule(ctx.orderId),
      compensate: async (ctx) => shippingService.cancel(ctx.orderId)
    }
  ];

  async run(context: OrderContext): Promise<void> {
    const completed: SagaStep[] = [];

    for (const step of this.steps) {
      try {
        await step.execute(context);
        completed.push(step);
      } catch (error) {
        console.error(`Saga step "${step.name}" failed, compensating...`);
        // Undo everything that succeeded so far, in reverse order
        for (const done of completed.reverse()) {
          await done.compensate(context);
        }
        throw error;
      }
    }
  }
}

declare const paymentService: { charge: Function; refund: Function };
declare const inventoryService: { reserve: Function; release: Function };
declare const shippingService: { schedule: Function; cancel: Function };
```

## Choreography Implementation

```typescript
// No orchestrator — each service only knows the event it listens for and the event it emits
class PaymentService {
  constructor(private eventBus: EventBus) {
    eventBus.subscribe('order.created', async (event) => {
      try {
        await this.charge(event.customerId, event.amount);
        await eventBus.publish('payment.processed', { orderId: event.orderId });
      } catch {
        await eventBus.publish('payment.failed', { orderId: event.orderId });
      }
    });
  }

  private async charge(customerId: string, amount: number): Promise<void> { /* ... */ }
}

class InventoryService {
  constructor(private eventBus: EventBus) {
    eventBus.subscribe('payment.processed', async (event) => {
      try {
        await this.reserve(event.orderId);
        await eventBus.publish('inventory.reserved', { orderId: event.orderId });
      } catch {
        // Compensate the step that came before this one
        await eventBus.publish('inventory.reservation-failed', { orderId: event.orderId });
      }
    });

    // Listens for a failure further down the chain to compensate its own action
    eventBus.subscribe('shipping.scheduling-failed', async (event) => {
      await this.release(event.orderId);
    });
  }

  private async reserve(orderId: string): Promise<void> { /* ... */ }
  private async release(orderId: string): Promise<void> { /* ... */ }
}

interface EventBus {
  subscribe(topic: string, handler: (event: any) => Promise<void>): void;
  publish(topic: string, event: any): Promise<void>;
}
```

## Comparison

| Aspect | Orchestration | Choreography |
|---|---|---|
| Process visibility | Explicit, lives in one place | Implicit, scattered across services' event handlers |
| Coupling | Services coupled to the orchestrator | Services coupled to event contracts, not to each other |
| Adding a new step | Change the orchestrator | Add a new subscriber, no existing service changes |
| Failure handling | Centralized compensation logic | Each service compensates its own step reactively |
| Debugging | Easier — one place to trace the whole flow | Harder — must reconstruct the flow from distributed event logs |
| Risk of a "god" component | The orchestrator can grow into a bottleneck | Risk of implicit, undocumented cross-service coupling |
| Best fit | Complex processes needing clear ownership and visibility | Simple, loosely coupled chains of reactions |

## Advantages

- **Orchestration — Visibility:** The entire process is readable in one place, making it easy to reason about and modify
- **Orchestration — Centralized Compensation:** Failure handling and rollback logic live together, not scattered
- **Choreography — Loose Coupling:** Services don't need to know about a central coordinator, only about events
- **Choreography — Easy Extension:** New reactions can be added by subscribing to existing events, with zero changes to existing services

## Disadvantages

- **Orchestration — Central Point of Coupling:** Every service in the process is coupled to the orchestrator; it can become a bottleneck or single point of failure
- **Orchestration — Orchestrator Complexity:** As steps grow, the orchestrator itself risks becoming a monolith of process logic
- **Choreography — Hidden Process Flow:** No single place shows the whole business process; understanding it means tracing events across many services
- **Choreography — Harder Debugging:** Diagnosing a stuck or failed process requires correlating logs across every participating service

## When to Use

- **Orchestration:** Complex processes with many steps, strict ordering, or where a clear audit of "what happened and in what order" is required
- **Orchestration:** Teams that need centralized control over retries, timeouts, and compensation logic
- **Choreography:** Simple, linear reaction chains where services are meant to evolve independently
- **Choreography:** Systems already built around [Event-Driven Architecture](../../arhitectures/event-driven-architecture/event-driven-architecture.md) where publishing events is already the norm

## Best Practices

- **Make Every Step Idempotent:** Both strategies rely on retries; steps must be safe to repeat
- **Design Compensations Up Front:** Don't add compensation logic as an afterthought — define it alongside the forward action
- **Correlate with a Saga/Trace ID:** Attach one ID to every event/call in a process for tracing, regardless of strategy
- **Don't Mix Silently:** If a system uses both strategies in different processes, document which one governs which process
- **Keep Orchestrators Thin:** An orchestrator should sequence calls, not contain business logic that belongs in the services it calls

## Related Patterns

- [Microservices Architecture](../../arhitectures/microservices-architecture/microservices-architecture.md) — the Saga Pattern this page describes is commonly used to coordinate cross-service transactions
- [Event-Driven Architecture](../../arhitectures/event-driven-architecture/event-driven-architecture.md) — choreography's underlying communication style
- [Publish-Subscribe Pattern](../pub-sub-pattern/pub-sub-pattern.md) — the mechanism choreography uses to propagate events
- [Service-Oriented Architecture](../../arhitectures/service-oriented-architecture/service-oriented-architecture.md) — orchestration originates from BPEL-style process coordination in classic SOA

## References & Sources

- Chris Richardson — Saga Pattern: https://microservices.io/patterns/data/saga.html
- Chris Richardson — "Microservices Patterns" (2018)
- Martin Fowler — Orchestration vs. Choreography discussion in microservices context: https://martinfowler.com/articles/microservices.html
- AWS — Step Functions (orchestration-based coordination): https://aws.amazon.com/step-functions/
