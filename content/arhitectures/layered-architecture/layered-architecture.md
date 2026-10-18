# Layered Architecture (N-Tier)

Layered Architecture organizes an application into horizontal layers stacked on top of each other, where each layer has a specific responsibility and (in its strict form) only talks to the layer directly beneath it. It's the oldest and most common architectural style — presentation on top, business logic in the middle, data access and storage at the bottom — and the baseline that styles like Clean, Hexagonal, and Onion later refine by inverting the dependency direction.

## Key Concepts

- **Layer:** A horizontal slice of responsibility (presentation, business/application, persistence, database)
- **Strict Layering:** A layer may only call the layer directly below it
- **Relaxed Layering:** A layer may call any layer below it, skipping intermediate ones
- **Open vs. Closed Layer:** A closed layer forces every request through it; an open layer can be bypassed for performance-sensitive paths
- **N-Tier:** Physical deployment split of the layers across separate processes or machines (presentation tier, application tier, database tier)
- **Downward Dependency:** Unlike Clean/Hexagonal, dependencies point straight down through concrete layers — the business layer depends directly on the data layer, not on an abstraction of it

## Architecture Layers

```
┌─────────────────────────────────────┐
│      Presentation Layer              │  UI, REST controllers, view models
├─────────────────────────────────────┤
│      Business Logic Layer            │  Services, validation, orchestration
├─────────────────────────────────────┤
│      Persistence Layer               │  Repositories, DAOs, ORM mappings
├─────────────────────────────────────┤
│      Database Layer                  │  The actual data store
└─────────────────────────────────────┘
        ▼ each layer only calls the one directly below (strict layering)
```

## Project Structure

```
src/
├── presentation/
│   ├── controllers/
│   │   ├── order.controller.ts
│   │   └── user.controller.ts
│   └── routes/
│       └── order.routes.ts
├── business/
│   ├── services/
│   │   ├── order.service.ts
│   │   └── pricing.service.ts
│   └── validators/
│       └── order.validator.ts
├── persistence/
│   ├── repositories/
│   │   ├── order.repository.ts
│   │   └── user.repository.ts
│   └── entities/
│       ├── order.entity.ts
│       └── user.entity.ts
└── database/
    ├── connection.ts
    └── migrations/
```

## Layer Implementation

### Presentation Layer

```typescript
// presentation/controllers/order.controller.ts
import { Router, Request, Response } from 'express';
import { OrderService } from '../../business/services/order.service';

export class OrderController {
  constructor(private orderService: OrderService) {}

  registerRoutes(router: Router): void {
    router.post('/orders', (req: Request, res: Response) => this.create(req, res));
    router.get('/orders/:id', (req: Request, res: Response) => this.getById(req, res));
  }

  private async create(req: Request, res: Response): Promise<void> {
    const order = await this.orderService.createOrder(req.body);
    res.status(201).json(order);
  }

  private async getById(req: Request, res: Response): Promise<void> {
    const order = await this.orderService.getOrder(req.params.id);
    res.json(order);
  }
}
```

### Business Logic Layer

```typescript
// business/services/order.service.ts
import { OrderRepository } from '../../persistence/repositories/order.repository';
import { PricingService } from './pricing.service';

export interface CreateOrderRequest {
  customerId: string;
  items: Array<{ productId: string; quantity: number; unitPrice: number }>;
}

// Business layer depends directly on the concrete repository below it —
// this is the key difference from Clean/Hexagonal, which would depend on an interface.
export class OrderService {
  constructor(
    private orderRepository: OrderRepository,
    private pricingService: PricingService
  ) {}

  async createOrder(request: CreateOrderRequest): Promise<Order> {
    if (request.items.length === 0) {
      throw new Error('Order must have at least one item');
    }

    const total = this.pricingService.calculateTotal(request.items);

    const order: Order = {
      id: crypto.randomUUID(),
      customerId: request.customerId,
      items: request.items,
      total,
      status: 'pending',
      createdAt: new Date()
    };

    return this.orderRepository.save(order);
  }

  async getOrder(id: string): Promise<Order | null> {
    return this.orderRepository.findById(id);
  }
}
```

### Persistence Layer

```typescript
// persistence/repositories/order.repository.ts
import { Database } from '../../database/connection';

export class OrderRepository {
  constructor(private db: Database) {}

  async save(order: Order): Promise<Order> {
    await this.db.query(
      `INSERT INTO orders (id, customer_id, total, status, created_at)
       VALUES ($1, $2, $3, $4, $5)`,
      [order.id, order.customerId, order.total, order.status, order.createdAt]
    );
    return order;
  }

  async findById(id: string): Promise<Order | null> {
    const row = await this.db.query('SELECT * FROM orders WHERE id = $1', [id]);
    return row ?? null;
  }
}
```

## Advantages

- **Simplicity:** Familiar, easy to explain to any team regardless of experience
- **Separation by Concern:** Presentation, logic, and data access are clearly split
- **Reusable Layers:** A single business layer can serve multiple presentation layers (web, mobile, CLI)
- **Testable in Isolation:** Each layer can be tested with the layer below mocked

## Disadvantages

- **Tight Coupling to Concretions:** Business logic depends directly on persistence, making it harder to swap databases than in inversion-based styles
- **Ripple Changes:** A change to the data model often forces changes up through business and presentation layers
- **Performance Overhead:** Strict layering means every call passes through every layer, even when a shortcut would be safe
- **Monolithic Tendency:** Layers naturally grow into one deployable unit, resisting independent scaling

## When to Use

- Small to medium applications with straightforward CRUD-heavy logic
- Teams new to architecture patterns who need a familiar mental model
- Systems where the database technology is unlikely to change
- Internal tools and admin panels where flexibility matters less than delivery speed

## Best Practices

- **Keep Layers Thin:** Push logic into the layer it belongs to instead of leaking it upward or downward
- **One-Way Dependencies:** Never let a lower layer call back up into a higher one
- **DTOs at Boundaries:** Don't leak persistence entities directly into the presentation layer
- **Consider Inversion Early:** If the database or framework is likely to change, prefer Hexagonal/Clean from the start instead of retrofitting later

## Common Mistakes

- **Leaky Abstractions:** Exposing ORM entities directly to controllers, coupling the API shape to the database schema
- **Fat Service Layer:** Dumping unrelated responsibilities into one "business layer" instead of splitting by domain
- **Skipping Layers Inconsistently:** Mixing strict and relaxed layering without a documented rule, making the codebase unpredictable
- **Treating Layers as Modules:** Layering is not the same as feature isolation — see [Feature-Based Architecture](../feature-based-architecture/feature-based-architecture.md) for the orthogonal concern

## Related Patterns

- [Clean Architecture](../clean-architecture/clean-architecture.md) — same horizontal idea, but dependencies point inward toward abstractions instead of straight down
- [Hexagonal Architecture](../hexagonal-architecture/hexagonal-architecture.md) — replaces direct layer-to-layer calls with ports and adapters
- [Separation of Concerns](../separation-of-concerns/separation-of-concerns.md) — the principle layered architecture is a direct application of
- [Repository Pattern](../../patterns/repository-pattern/repository-pattern.md) — typical persistence-layer abstraction

## References & Sources

- Martin Fowler — "Patterns of Enterprise Application Architecture" (2002), Chapter on Layering
- Microsoft — N-Tier Architecture Style: https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/n-tier
- Mark Richards & Neal Ford — "Fundamentals of Software Architecture" (2020)
