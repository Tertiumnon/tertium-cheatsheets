# Spec-Driven Architecture

Spec-Driven Architecture organizes development around a written, structured specification that precedes and governs implementation. The spec — not a ticket title or a prompt — is the source of truth: requirements, contracts, and acceptance criteria are captured first, then code, tests, and documentation are derived from it. This matters most when a human or an AI agent implements a feature without full context: an unambiguous spec constrains the solution space before a single line of code is written, instead of relying on review to catch drift after the fact.

## Key Concepts

- **Specification:** A structured, versioned document describing what a feature must do — not how. Lives in the repo, reviewed like code.
- **Contract:** The machine-checkable shape of an interface (API schema, type definitions, event payloads) derived from the spec.
- **Acceptance Criteria:** Concrete, testable conditions that define "done" — usually Given/When/Then or a checklist.
- **Plan:** The technical approach chosen to satisfy the spec (architecture, data model, sequencing) — separate from the spec itself so "what" and "how" don't get tangled.
- **Traceability:** Every requirement in the spec maps to at least one test and one implementation unit; every implementation unit maps back to a requirement.
- **Living Document:** The spec is updated when reality changes, instead of rotting the moment code diverges from it.

## Development Workflow

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Specify   │───▶│    Plan     │───▶│    Tasks    │───▶│  Implement  │
│  (what/why) │    │  (how/arch) │    │ (unit steps)│    │  + verify   │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
       │                                                          │
       └──────────────────── feedback / spec update ◀─────────────┘
```

- **Specify:** Capture intent, constraints, and acceptance criteria — no implementation detail.
- **Plan:** Choose the technical approach: data model, layers touched, dependencies, edge cases.
- **Tasks:** Break the plan into small, independently verifiable units of work.
- **Implement + verify:** Write code and tests against the spec's acceptance criteria; if reality forces a change, the spec is updated, not silently abandoned.

## Project Structure

```
specs/
├── 001-checkout-flow/
│   ├── spec.md                  # What & why: requirements, acceptance criteria
│   ├── plan.md                  # How: architecture, data model, sequencing
│   ├── tasks.md                 # Ordered, checkable implementation steps
│   └── contracts/
│       ├── checkout.openapi.yaml
│       └── checkout-events.schema.json
├── 002-refund-policy/
│   ├── spec.md
│   ├── plan.md
│   └── tasks.md
└── constitution.md              # Project-wide non-negotiables (see below)
```

## Writing a Specification

A spec stays implementation-agnostic and testable — it describes behavior, not code:

```markdown
# Spec 001: Checkout Flow

## Problem
Users abandon checkout when shipping cost only appears at the final step.

## Requirements
- MUST show shipping cost before payment method is selected
- MUST support at least one guest checkout path (no account required)
- MUST NOT allow order submission with an empty cart

## Acceptance Criteria
1. Given a cart with items, when the user reaches step 2,
   then shipping cost is visible before any payment field renders.
2. Given no active session, when the user starts checkout,
   then a guest checkout option is offered.
3. Given an empty cart, when the user requests checkout,
   then the system rejects the request with a clear error.

## Out of Scope
- Multi-currency pricing (tracked separately in spec 004)
```

Requirement keywords (`MUST`, `MUST NOT`, `SHOULD`) follow RFC 2119 so priority is unambiguous — a reviewer or an AI agent can tell a hard constraint from a preference at a glance.

## From Spec to Contract

Contracts make the spec machine-checkable instead of just human-readable:

```yaml
# specs/001-checkout-flow/contracts/checkout.openapi.yaml
paths:
  /checkout:
    post:
      requestBody:
        content:
          application/json:
            schema:
              type: object
              required: [cartId, shippingAddress]
              properties:
                cartId: { type: string }
                shippingAddress: { $ref: '#/components/schemas/Address' }
      responses:
        '200':
          description: Checkout session created with shipping cost included
        '422':
          description: Cart is empty
```

```typescript
// Generated from the contract — implementation code depends on this, not the other way around
interface CheckoutRequest {
  cartId: string;
  shippingAddress: Address;
}

interface CheckoutResponse {
  sessionId: string;
  shippingCost: Money;
  paymentOptions: PaymentMethod[];
}

class CheckoutService {
  async startCheckout(req: CheckoutRequest): Promise<CheckoutResponse> {
    const cart = await this.cartRepository.findById(req.cartId);
    if (!cart || cart.items.length === 0) {
      throw new EmptyCartError(req.cartId);
    }
    const shippingCost = await this.shippingService.quote(cart, req.shippingAddress);
    return {
      sessionId: generateId(),
      shippingCost,
      paymentOptions: await this.paymentService.availableMethods(req.shippingAddress),
    };
  }
}
```

## From Spec to Tests

Acceptance criteria translate directly into executable tests — no separate interpretation step:

```typescript
// specs/001-checkout-flow acceptance criteria → tests
describe('Spec 001: Checkout Flow', () => {
  it('AC1: shows shipping cost before payment method is selected', async () => {
    const cart = givenCartWithItems([{ productId: 'p1', qty: 1 }]);
    const response = await checkoutService.startCheckout({ cartId: cart.id, shippingAddress: aValidAddress() });
    expect(response.shippingCost).toBeDefined();
    expect(response.paymentOptions.length).toBeGreaterThan(0);
  });

  it('AC2: offers guest checkout when no session exists', async () => {
    const response = await checkoutService.startCheckout({ cartId: guestCart.id, shippingAddress: aValidAddress() });
    expect(response.sessionId).toBeDefined();
  });

  it('AC3: rejects checkout with an empty cart', async () => {
    const emptyCart = givenCartWithItems([]);
    await expect(
      checkoutService.startCheckout({ cartId: emptyCart.id, shippingAddress: aValidAddress() })
    ).rejects.toThrow(EmptyCartError);
  });
});
```

## Spec-Driven Development with AI Agents

When an AI agent writes the implementation, the spec is the mechanism that constrains it *before* generation instead of only catching drift in review:

```
constitution.md   → project-wide invariants ("MUST NOT bypass the repository layer")
spec.md           → what this feature must do, in this PR's scope
plan.md           → the approach the agent commits to before writing code
tasks.md          → small steps, each independently reviewable
   │
   ▼
Agent implements task-by-task, checked against spec.md's acceptance criteria
   │
   ▼
TypeScript strict mode → tests → human code review   (see best-practices/code-review)
```

A vague prompt ("add checkout") leaves the agent to infer requirements, and inference is where hallucinated edge cases and silently-dropped requirements come from. A spec with explicit `MUST`/`MUST NOT` requirements and Given/When/Then acceptance criteria removes that inference step — the agent implements against a fixed target, and the same spec becomes the review checklist afterward.

## Advantages

- **Shared Understanding:** Product, engineering, and AI agents work from the same unambiguous document instead of a chain of paraphrased tickets.
- **Reduced Rework:** Ambiguity is resolved before implementation, not after a review round-trip.
- **Traceability:** Every requirement maps to a test; every test maps to a requirement — coverage gaps are visible.
- **AI-Agent Alignment:** Explicit constraints reduce hallucinated scope and silently-dropped edge cases in generated code.
- **Onboarding:** New contributors read the spec, not archaeology through commit history, to understand intent.

## Disadvantages

- **Upfront Cost:** Writing a good spec takes real time; trivial changes don't justify the ceremony.
- **Spec Rot:** A spec that isn't updated when reality changes becomes actively misleading — worse than no spec.
- **False Precision:** Overly detailed specs can drift into prescribing implementation, defeating the separation of "what" from "how."
- **Process Overhead:** Adds a review/approval step before implementation can start, which can slow fast-moving exploratory work.

## When to Use

- Features with non-obvious edge cases or cross-team/cross-service contracts
- Work delegated to an AI coding agent, where ambiguity directly becomes implementation risk
- APIs and event schemas consumed by other teams or external clients
- Regulated domains where acceptance criteria must be auditable
- Large or long-lived features where "why" needs to survive past the original author

## Best Practices

- **Separate What from How:** Keep `spec.md` free of implementation detail; put architecture decisions in `plan.md`.
- **Testable Criteria:** Every acceptance criterion should be phrased so a test can pass or fail against it directly.
- **Explicit Priority:** Use `MUST` / `SHOULD` / `MAY` (RFC 2119) so agents and reviewers don't guess at severity.
- **Scope Boundaries:** State what's explicitly out of scope to stop scope creep during implementation.
- **Version Specs with Code:** Store specs in the repo, reviewed and diffed like any other change.
- **Update, Don't Abandon:** When implementation reveals the spec was wrong, edit the spec in the same PR.

## Common Mistakes

- **Prescribing Implementation:** Spec dictates class names or a specific library instead of behavior.
- **Untestable Criteria:** Vague acceptance criteria like "should be fast" or "should be intuitive" with no measurable condition.
- **Write-Once Specs:** Treating the spec as a one-time artifact instead of keeping it in sync with the shipped behavior.
- **Skipping the Plan Step:** Jumping from spec straight to code hides architectural decisions that should have been reviewed first.
- **Over-Specifying Trivial Work:** Applying full spec ceremony to a one-line bug fix.

## Related Patterns

- [Domain-Driven Architecture](../arhitectures/domain-driven-architecture/domain-driven-architecture.md) — shared vocabulary (ubiquitous language) plays the same role as a spec's shared requirements
- [Testing Strategy](../testing/testing-strategy.md) — acceptance criteria in a spec become the test suite's backbone
- [Code Review Guidelines](../best-practices/code-review/code-review.md) — the spec is the checklist a reviewer verifies generated code against

## References & Sources

- OpenAPI Specification: https://swagger.io/specification/
- Gherkin Reference (Cucumber): https://cucumber.io/docs/gherkin/
- Martin Fowler — Specification by Example: https://martinfowler.com/bliki/SpecificationByExample.html
- RFC 2119 — Key words for use in RFCs to Indicate Requirement Levels: https://www.rfc-editor.org/rfc/rfc2119
- Gojko Adzic — "Specification by Example: How Successful Teams Deliver the Right Software" (2011)
