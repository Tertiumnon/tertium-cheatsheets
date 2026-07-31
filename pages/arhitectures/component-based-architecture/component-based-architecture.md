# Component-Based Architecture

Component-Based Architecture builds a system from loosely-coupled, independently replaceable components, each exposing a well-defined interface and hiding its internal implementation. Components typically run within the same process and communicate through direct interface calls rather than the network — the same idea as Service-Oriented Architecture, but scoped to a single application instead of distributed across machines.

## Key Concepts

- **Component:** A self-contained unit of functionality exposing a public interface and hiding internal state
- **Interface/Contract:** The only way a component may be accessed — implementation details stay private
- **Composition:** Larger behavior is built by wiring components together, not by inheritance
- **Encapsulation:** A component can be replaced entirely as long as its interface contract is honored
- **Reusability:** Components are designed to be reused across features or even across applications
- **Extension Point / Plug-in:** A defined slot where new components can be registered without modifying existing code
- **Component Registry:** Central lookup that resolves interfaces to concrete implementations at runtime

## Architecture Diagram

```
┌───────────────────────────────────────────────┐
│                 Application                    │
│                                                 │
│   ┌───────────┐   uses    ┌────────────────┐  │
│   │  Client   │──────────▶│ IPaymentMethod │  │  ◀── interface (contract)
│   └───────────┘           └────────┬───────┘  │
│                                     │           │
│                     ┌───────────────┼───────┐   │
│                     ▼               ▼       ▼   │
│              ┌──────────┐   ┌──────────┐ ┌────┐ │
│              │CreditCard│   │  PayPal  │ │Wire│ │  ◀── interchangeable components
│              └──────────┘   └──────────┘ └────┘ │
└───────────────────────────────────────────────┘
```

## Example: Interface-Driven Components

```typescript
// Contract every component must satisfy
interface NotificationChannel {
  send(recipient: string, message: string): Promise<void>;
}

// Independently replaceable components
class EmailChannel implements NotificationChannel {
  async send(recipient: string, message: string): Promise<void> {
    console.log(`Emailing ${recipient}: ${message}`);
  }
}

class SmsChannel implements NotificationChannel {
  async send(recipient: string, message: string): Promise<void> {
    console.log(`Texting ${recipient}: ${message}`);
  }
}

class PushChannel implements NotificationChannel {
  async send(recipient: string, message: string): Promise<void> {
    console.log(`Push notification to ${recipient}: ${message}`);
  }
}

// Composition root wires components together; nothing else knows the concrete types
class NotificationService {
  constructor(private channels: NotificationChannel[]) {}

  async notify(recipient: string, message: string): Promise<void> {
    await Promise.all(this.channels.map(channel => channel.send(recipient, message)));
  }
}

// Usage
const notificationService = new NotificationService([
  new EmailChannel(),
  new SmsChannel()
]);
await notificationService.notify('user@example.com', 'Your order has shipped');
```

## Example: Plugin Architecture with Dynamic Registration

```typescript
interface PricingPlugin {
  readonly name: string;
  apply(basePrice: number, context: PricingContext): number;
}

interface PricingContext {
  customerTier: 'standard' | 'premium';
  quantity: number;
}

class ComponentRegistry<T extends { name: string }> {
  private components = new Map<string, T>();

  register(component: T): void {
    this.components.set(component.name, component);
  }

  get(name: string): T {
    const component = this.components.get(name);
    if (!component) throw new Error(`Component not registered: ${name}`);
    return component;
  }

  getAll(): T[] {
    return [...this.components.values()];
  }
}

class VolumeDiscountPlugin implements PricingPlugin {
  readonly name = 'volume-discount';
  apply(basePrice: number, context: PricingContext): number {
    return context.quantity >= 10 ? basePrice * 0.9 : basePrice;
  }
}

class PremiumTierPlugin implements PricingPlugin {
  readonly name = 'premium-tier';
  apply(basePrice: number, context: PricingContext): number {
    return context.customerTier === 'premium' ? basePrice * 0.95 : basePrice;
  }
}

// New pricing rules register themselves without touching PricingEngine's code
const registry = new ComponentRegistry<PricingPlugin>();
registry.register(new VolumeDiscountPlugin());
registry.register(new PremiumTierPlugin());

class PricingEngine {
  constructor(private registry: ComponentRegistry<PricingPlugin>) {}

  calculate(basePrice: number, context: PricingContext): number {
    return this.registry.getAll().reduce((price, plugin) => plugin.apply(price, context), basePrice);
  }
}
```

## Advantages

- **Independent Replacement:** Swap a component's implementation without touching callers, as long as the interface holds
- **Parallel Development:** Teams build components against an agreed interface before the implementation exists
- **Testability:** Components are trivially mocked behind their interface
- **Reduced Duplication:** Shared components are reused instead of reimplemented per feature

## Disadvantages

- **Interface Design Cost:** Getting the contract right upfront is hard; a wrong interface ripples through every implementation
- **Indirection Overhead:** More interfaces and registries to navigate than a straightforward concrete call
- **In-Process Only:** Doesn't address network communication, deployment independence, or scaling — that's [Service-Oriented Architecture](../service-oriented-architecture/service-oriented-architecture.md) or [Microservices](../microservices-architecture/microservices-architecture.md)
- **Versioning Friction:** Changing a widely-used interface requires coordinating every implementation at once

## When to Use

- Applications with interchangeable strategies (payment methods, pricing rules, notification channels)
- Plugin or extension systems where third parties add behavior without modifying core code
- Codebases where the same functional unit needs to be reused across multiple features
- UI component libraries and design systems

## Best Practices

- **Design the Interface First:** Agree on the contract before writing implementations
- **Keep Interfaces Narrow:** Small, focused interfaces are easier to implement and mock than large ones
- **Favor Composition Over Inheritance:** Wire components together rather than extending base classes
- **Version Contracts Explicitly:** Treat breaking interface changes like breaking API changes
- **Centralize Wiring:** Keep component registration/composition in one place (a composition root), not scattered

## Related Patterns

- [Service-Oriented Architecture](../service-oriented-architecture/service-oriented-architecture.md) — the same interchangeable-unit idea, scaled across the network
- [Strategy Pattern](../../patterns/strategy-pattern/strategy-pattern.md) — the design-pattern-level version of a swappable component
- [Dependency Injection Pattern](../../patterns/dependency-injection-pattern/dependency-injection.md) — the mechanism typically used to wire components together
- [Factory Pattern](../../patterns/factory-pattern/factory-pattern.md) — used to construct the correct component implementation at runtime

## References & Sources

- Clemens Szyperski — "Component Software: Beyond Object-Oriented Programming" (2002)
- Martin Fowler — Inversion of Control Containers and the Dependency Injection Pattern: https://martinfowler.com/articles/injection.html
- OSGi Alliance — Component-Based Architecture for Java: https://www.osgi.org/
