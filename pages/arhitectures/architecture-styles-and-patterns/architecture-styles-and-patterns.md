# Architectural Styles vs. Architectural Patterns

Architectural style and architectural pattern are often used interchangeably, but they operate at different altitudes. A **style** is a broad, high-level strategy for organizing an entire system — wide brush strokes about how components are structured and how they communicate. A **pattern** is a concrete, problem-specific solution applied inside (or across) that structure — intricate details for solving one recurring problem, like how to coordinate a multi-service transaction or how to keep reads fast while writes stay consistent. A single system typically picks one or two styles and then applies several patterns within them.

## Core Distinction

- **Scope:** A style shapes the whole system (how it's decomposed, deployed, and where boundaries sit). A pattern solves one recurring problem within that shape.
- **Granularity:** Styles are wide brush strokes — "services communicate over the network," "layers stack downward." Patterns are intricate details — "here's exactly how a command becomes a read model."
- **Composability:** Multiple patterns typically live inside one style. Microservices (style) commonly uses CQRS, Event Sourcing, and Saga orchestration/choreography (patterns) together.
- **Reusability across styles:** The same pattern often applies under different styles — CQRS shows up in monoliths, microservices, and event-driven systems alike.
- **Decision order:** Style is usually chosen first (how the system is shaped), patterns are chosen per problem as they arise during design and implementation.

## The Ten Major Architectural Styles

1. **[Layered / N-Tier](../layered-architecture/layered-architecture.md)** — Horizontal layers (presentation, business logic, data) where each layer depends only on the one below it.
2. **[Component-Based](../component-based-architecture/component-based-architecture.md)** — Loosely-coupled, independently replaceable components composed through well-defined interfaces, usually within one process.
3. **[Service-Oriented (SOA)](../service-oriented-architecture/service-oriented-architecture.md)** — Coarse-grained, reusable services integrated through a shared bus and shared contracts.
4. **[Distributed Systems](../distributed-architecture/distributed-architecture.md)** — Multiple networked computers cooperating to behave as one coherent system, dealing with partial failure and consistency.
5. **[Domain-Driven](../domain-driven-architecture/domain-driven-architecture.md)** — Structure centers on the business domain and a shared ubiquitous language, organized into bounded contexts.
6. **[Event-Driven](../event-driven-architecture/event-driven-architecture.md)** — Events (state changes) are the primary driver of communication and control flow.
7. **[Separation of Concerns](../separation-of-concerns/separation-of-concerns.md)** — Functionality is divided into independent sections, each addressing one concern; the principle underlying most other styles.
8. **[Interpreter](../interpreter-architecture/interpreter-architecture.md)** — Instructions (a DSL, rules, config) are parsed and executed directly at runtime instead of being pre-compiled.
9. **[Concurrency](../concurrency-architecture/concurrency-architecture.md)** — The system is organized around executing independent tasks simultaneously or in overlapping time.
10. **[Data-Centric](../data-centric-architecture/data-centric-architecture.md)** — A shared, persistent data store (or blackboard) is the hub that independent components read from and write to.

Real systems mix these. A microservices deployment (distributed + event-driven) is typically domain-driven internally, with each service applying separation of concerns and, in places, concurrency or interpreter techniques.

## Key Architectural Patterns

- **[Clean Architecture](../clean-architecture/clean-architecture.md) / [Onion Architecture](../clean-architecture/clean-architecture--onion.md)** — Concentric layers with dependencies pointing inward toward a framework-independent domain core.
- **[Hexagonal Architecture (Ports & Adapters)](../hexagonal-architecture/hexagonal-architecture.md)** — Isolates the core via ports (contracts) and adapters (implementations), the same dependency-inversion idea as Clean/Onion with a different vocabulary.
- **[Microservices](../microservices-architecture/microservices-architecture.md)** — Concrete application of the Service-Oriented and Distributed styles: small, independently deployable services, each owning its data.
- **[Publish-Subscribe](../../patterns/pub-sub-pattern/pub-sub-pattern.md)** — Broker-mediated decoupling of publishers and subscribers, the messaging backbone of most event-driven systems.
- **[CQRS](../../patterns/cqrs-pattern/cqrs-pattern.md)** — Separates the write model (commands) from the read model (queries), often with different storage optimized for each.
- **[Event Sourcing](../../patterns/event-sourcing-pattern/event-sourcing-pattern.md)** — Persists state as an immutable sequence of events instead of current-state snapshots; state is derived by replay.
- **[Orchestration vs. Choreography](../../patterns/orchestration-choreography-pattern/orchestration-choreography-pattern.md)** — Two opposing strategies for coordinating a multi-service business process (central coordinator vs. reactive event chain).

## Choosing Between Style and Pattern

| Question | Answer points to |
|---|---|
| "How should the whole system be shaped and deployed?" | Architectural style |
| "How do I keep this one process's read and write paths from fighting each other?" | Architectural pattern (CQRS) |
| "Should services be many-and-small or few-and-coarse?" | Architectural style (Microservices vs. SOA) |
| "How do I coordinate this one multi-step transaction across services?" | Architectural pattern (Orchestration/Choreography) |
| "Where does business logic live relative to frameworks and databases?" | Architectural pattern (Clean/Onion/Hexagonal) applied within a style |

## Related

- [Domain-Driven Architecture](../domain-driven-architecture/domain-driven-architecture.md)
- [Event-Driven Architecture](../event-driven-architecture/event-driven-architecture.md)
- [Microservices Architecture](../microservices-architecture/microservices-architecture.md)
- [Feature-Based Architecture](../feature-based-architecture/feature-based-architecture.md)
- [Spec-Driven Architecture](../spec-driven-architecture/spec-driven-architecture.md)

## References & Sources

- Mark Richards & Neal Ford — "Fundamentals of Software Architecture" (2020)
- Martin Fowler — Software Architecture Guide: https://martinfowler.com/architecture/
- Microsoft Azure — Architecture Styles: https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/
- Gang of Four — "Design Patterns: Elements of Reusable Object-Oriented Software" (1994)
