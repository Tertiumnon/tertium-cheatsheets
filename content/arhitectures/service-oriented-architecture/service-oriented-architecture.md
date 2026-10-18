# Service-Oriented Architecture (SOA)

Service-Oriented Architecture structures an application as a set of coarse-grained, reusable business services that communicate over a network, typically through a shared integration layer such as an Enterprise Service Bus (ESB). SOA is the direct predecessor of microservices: where microservices favor many small, independently deployable services with decentralized data, classic SOA favors fewer, larger services integrated through a centralized bus with shared contracts and, often, a shared database.

## Key Concepts

- **Service:** A coarse-grained, reusable unit exposing business capability through a formal contract (WSDL/SOAP or a versioned REST/OpenAPI contract)
- **Enterprise Service Bus (ESB):** Centralized middleware that routes, transforms, and orchestrates messages between services
- **Service Contract:** A formally defined, versioned interface consumers code against, independent of implementation
- **Service Registry:** Directory where services publish their contracts and locations for discovery (historically UDDI)
- **Orchestration:** A central process (often BPEL) drives the sequence of service calls for a business process
- **Loose Coupling:** Services interact only through contracts, never through internal implementation details
- **Interoperability:** Services are built to be consumed regardless of the caller's platform or language
- **Shared Data Model:** Unlike microservices, SOA commonly standardizes a canonical data model shared across services

## Architecture Diagram

```
                 ┌──────────────────────────────────┐
                 │      Enterprise Service Bus       │
                 │  (Routing, Transformation,        │
                 │   Orchestration, Protocol Bridge)  │
                 └───┬─────────┬─────────┬─────────┬─┘
                     │         │         │         │
              ┌──────▼──┐ ┌───▼────┐ ┌──▼─────┐ ┌─▼──────┐
              │ Billing │ │ Order  │ │Customer│ │Shipping│
              │ Service │ │Service │ │Service │ │Service │
              └────┬────┘ └───┬────┘ └───┬────┘ └───┬────┘
                   └──────────┴──────────┴──────────┘
                          shared canonical database
```

Contrast with microservices' point-to-point, database-per-service topology in [Microservices Architecture](../microservices-architecture/microservices-architecture.md).

## Service Contract Example

```typescript
// contracts/order-service.contract.ts
// A versioned, formal contract — consumers depend on this, never on OrderService internals
export interface OrderServiceContractV1 {
  submitOrder(request: SubmitOrderRequest): Promise<SubmitOrderResponse>;
  getOrderStatus(orderId: string): Promise<OrderStatusResponse>;
}

export interface SubmitOrderRequest {
  customerId: string;
  lineItems: Array<{ sku: string; quantity: number }>;
}

export interface SubmitOrderResponse {
  orderId: string;
  estimatedTotal: number;
}
```

## ESB-Style Message Routing

```typescript
// esb/message-router.ts
interface EsbMessage {
  service: string;
  operation: string;
  payload: unknown;
  correlationId: string;
}

type ServiceHandler = (payload: unknown) => Promise<unknown>;

class EnterpriseServiceBus {
  private routes = new Map<string, ServiceHandler>();

  register(service: string, operation: string, handler: ServiceHandler): void {
    this.routes.set(`${service}.${operation}`, handler);
  }

  async route(message: EsbMessage): Promise<unknown> {
    const key = `${message.service}.${message.operation}`;
    const handler = this.routes.get(key);
    if (!handler) throw new Error(`No route registered for ${key}`);

    // Cross-cutting concerns handled centrally by the bus, not by each service
    console.log(`[ESB] Routing ${key} (correlationId: ${message.correlationId})`);
    return handler(message.payload);
  }
}

// Services register themselves with the bus instead of calling each other directly
const bus = new EnterpriseServiceBus();
bus.register('order', 'submit', async (payload) => {
  // OrderService business logic
  return { orderId: crypto.randomUUID(), estimatedTotal: 129.99 };
});

// A consumer only ever talks to the bus
const response = await bus.route({
  service: 'order',
  operation: 'submit',
  payload: { customerId: 'cust-1', lineItems: [{ sku: 'SKU-1', quantity: 2 }] },
  correlationId: crypto.randomUUID()
});
```

## Orchestrated Business Process

```typescript
// A central orchestrator drives a multi-service business process — see
// Orchestration vs. Choreography Pattern for the distributed-systems version of this idea.
class OrderFulfillmentOrchestrator {
  constructor(private bus: EnterpriseServiceBus) {}

  async fulfill(customerId: string, lineItems: Array<{ sku: string; quantity: number }>): Promise<void> {
    const order = await this.bus.route({
      service: 'order', operation: 'submit',
      payload: { customerId, lineItems }, correlationId: crypto.randomUUID()
    });

    await this.bus.route({
      service: 'billing', operation: 'charge',
      payload: order, correlationId: crypto.randomUUID()
    });

    await this.bus.route({
      service: 'shipping', operation: 'schedule',
      payload: order, correlationId: crypto.randomUUID()
    });
  }
}
```

## SOA vs. Microservices

| Aspect | SOA | Microservices |
|---|---|---|
| Service granularity | Coarse-grained | Fine-grained |
| Integration | Centralized ESB | Direct calls or lightweight event bus |
| Data | Often shared/canonical database | Database per service |
| Governance | Centralized contracts and standards | Decentralized, team-owned |
| Reuse goal | Maximize reuse across the enterprise | Maximize independent deployability |
| Typical protocol | SOAP/XML (historically), REST | REST, gRPC, events |

## Advantages

- **Enterprise Reuse:** Services are designed as shared business capabilities usable across many applications
- **Centralized Governance:** One place enforces contracts, security, and monitoring standards
- **Protocol Bridging:** ESB can mediate between heterogeneous systems (mainframes, SOAP, REST)
- **Formal Contracts:** Versioned contracts make integration explicit and auditable

## Disadvantages

- **ESB Bottleneck:** The bus becomes both a single point of failure and a performance chokepoint
- **Slow Change Velocity:** Shared canonical models and centralized governance slow down independent teams
- **Heavyweight Tooling:** Historically tied to complex standards (SOAP, WS-*, BPEL) with significant operational overhead
- **Coarse Granularity:** Larger services are harder to scale independently than microservices

## When to Use

- Large enterprises integrating many existing, heterogeneous systems (legacy mainframes, third-party platforms)
- Organizations that need centralized governance and auditability over service contracts
- Environments where a strong, dedicated integration team maintains the ESB
- B2B integration scenarios requiring standardized, versioned contracts across organizational boundaries

## Best Practices

- **Contract-First Design:** Define and version the service contract before implementation
- **Keep the Bus Thin Where Possible:** Avoid pushing business logic into ESB transformation rules
- **Explicit Versioning:** Never break an existing contract; publish a new version instead
- **Monitor the Bus:** The ESB is a critical dependency — instrument it as carefully as any service

## Related Patterns

- [Microservices Architecture](../microservices-architecture/microservices-architecture.md) — SOA's fine-grained, decentralized successor
- [Distributed Systems Architecture](../distributed-architecture/distributed-architecture.md) — the broader style SOA and microservices both operate within
- [Orchestration vs. Choreography Pattern](../../patterns/orchestration-choreography-pattern/orchestration-choreography-pattern.md) — coordination strategies used inside SOA and microservices alike
- [Component-Based Architecture](../component-based-architecture/component-based-architecture.md) — the same interchangeable-unit idea scoped to a single process

## References & Sources

- Thomas Erl — "SOA: Principles of Service Design" (2007)
- Martin Fowler — Microservices vs. SOA: https://martinfowler.com/articles/microservices.html
- Microsoft — Service-Oriented Architecture: https://learn.microsoft.com/en-us/previous-versions/msp-n-p/ee658117(v=pandp.10)
