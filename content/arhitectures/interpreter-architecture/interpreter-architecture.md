# Interpreter Architecture

An interpreter-style architecture executes instructions directly at runtime instead of compiling them ahead of time into machine code. The system reads a program — a domain-specific language, a rules document, a config-driven workflow — and walks through it step by step, evaluating each instruction against the current state. This is the architecture behind rule engines, expression evaluators, template engines, and any scripting layer embedded inside a larger application.

## Key Concepts

- **Grammar:** The formal rules describing what a valid instruction/expression looks like
- **Lexer/Tokenizer:** Breaks raw source text into a stream of tokens
- **Parser:** Turns the token stream into a structured representation, usually an Abstract Syntax Tree (AST)
- **AST (Abstract Syntax Tree):** A tree representation of the parsed instructions, ready to be evaluated
- **Interpreter/Evaluator:** Walks the AST and produces a result by executing each node against the current context
- **Context/Environment:** The state (variables, bindings) available to the interpreter while evaluating

## Architecture Diagram

```
Source Text ──▶ Lexer ──▶ Tokens ──▶ Parser ──▶ AST ──▶ Interpreter ──▶ Result
                                                            ▲
                                                            │
                                                       Context/Environment
                                                     (variables, functions)
```

## Example: Arithmetic Expression Interpreter

```typescript
// 1. Tokenizer
type Token = { type: 'NUMBER' | 'PLUS' | 'MINUS' | 'TIMES'; value: string };

function tokenize(input: string): Token[] {
  const tokens: Token[] = [];
  const regex = /\s*(\d+|\+|-|\*)\s*/g;
  let match: RegExpExecArray | null;
  while ((match = regex.exec(input))) {
    const value = match[1];
    if (/\d/.test(value)) tokens.push({ type: 'NUMBER', value });
    else if (value === '+') tokens.push({ type: 'PLUS', value });
    else if (value === '-') tokens.push({ type: 'MINUS', value });
    else if (value === '*') tokens.push({ type: 'TIMES', value });
  }
  return tokens;
}

// 2. AST node types
type Expr =
  | { kind: 'Number'; value: number }
  | { kind: 'BinaryOp'; op: '+' | '-' | '*'; left: Expr; right: Expr };

// 3. Parser (builds a left-to-right AST, no precedence handling for brevity)
function parse(tokens: Token[]): Expr {
  let position = 0;

  function parseNumber(): Expr {
    const token = tokens[position++];
    return { kind: 'Number', value: Number(token.value) };
  }

  let expr = parseNumber();
  while (position < tokens.length) {
    const opToken = tokens[position++];
    const op = opToken.type === 'PLUS' ? '+' : opToken.type === 'MINUS' ? '-' : '*';
    const right = parseNumber();
    expr = { kind: 'BinaryOp', op, left: expr, right };
  }
  return expr;
}

// 4. Interpreter (walks the AST and evaluates it directly — no compilation step)
function evaluate(node: Expr): number {
  switch (node.kind) {
    case 'Number':
      return node.value;
    case 'BinaryOp': {
      const left = evaluate(node.left);
      const right = evaluate(node.right);
      switch (node.op) {
        case '+': return left + right;
        case '-': return left - right;
        case '*': return left * right;
      }
    }
  }
}

// Usage
const ast = parse(tokenize('3 + 4 * 2'));
console.log(evaluate(ast)); // 14 (left-to-right, no precedence, by design of this minimal example)
```

## Example: Business Rule Engine

```typescript
// Rules are data, not code — the interpreter evaluates them against a context at runtime,
// which lets non-developers author and change business logic without a deployment.
interface Rule {
  field: string;
  operator: 'eq' | 'gt' | 'lt' | 'in';
  value: unknown;
}

interface RuleSet {
  all?: Rule[];
  any?: Rule[];
}

class RuleEngine {
  evaluate(ruleSet: RuleSet, context: Record<string, unknown>): boolean {
    if (ruleSet.all) return ruleSet.all.every(rule => this.evaluateRule(rule, context));
    if (ruleSet.any) return ruleSet.any.some(rule => this.evaluateRule(rule, context));
    return true;
  }

  private evaluateRule(rule: Rule, context: Record<string, unknown>): boolean {
    const actual = context[rule.field];
    switch (rule.operator) {
      case 'eq': return actual === rule.value;
      case 'gt': return (actual as number) > (rule.value as number);
      case 'lt': return (actual as number) < (rule.value as number);
      case 'in': return (rule.value as unknown[]).includes(actual);
    }
  }
}

// Loaded from a config file or database — no code change needed to update business logic
const freeShippingRule: RuleSet = {
  all: [
    { field: 'orderTotal', operator: 'gt', value: 50 },
    { field: 'country', operator: 'in', value: ['US', 'CA'] }
  ]
};

const engine = new RuleEngine();
const qualifies = engine.evaluate(freeShippingRule, { orderTotal: 75, country: 'US' });
console.log(qualifies); // true
```

## Advantages

- **Flexibility Without Redeployment:** Rules/scripts can change without recompiling or redeploying the host application
- **Non-Developer Authorship:** Business users can author rules through a DSL or config format
- **Sandboxing:** Interpreted logic can be tightly controlled and constrained, unlike arbitrary compiled code
- **Rapid Iteration:** No build step between changing logic and seeing the result

## Disadvantages

- **Runtime Performance:** Interpreting is slower than executing pre-compiled code, especially for tight loops
- **Limited Static Checking:** Errors in the interpreted program surface at runtime, not compile time
- **Grammar/Parser Maintenance:** The DSL's grammar and parser are extra code to design, test, and evolve
- **Debugging Difficulty:** Stepping through an interpreter loop is harder than stepping through native code

## When to Use

- Business rules that change frequently and shouldn't require a deployment
- Embedding a small scripting or expression language inside an application (formula fields, filters, templating)
- Configuration-driven workflows where non-developers define behavior
- Building domain-specific languages that express intent more directly than general-purpose code

## Best Practices

- **Keep the Grammar Small:** A minimal DSL is easier to implement correctly and to reason about than a general-purpose language
- **Validate Before Executing:** Parse and type-check the full program before evaluating any of it
- **Sandbox Side Effects:** Restrict what interpreted code can touch — no arbitrary file/network access unless explicitly required
- **Cache Parsed ASTs:** Avoid re-tokenizing and re-parsing the same rule on every evaluation
- **Prefer Data Over Code:** Represent rules as data (JSON/YAML) rather than a string of a real programming language when possible — it's safer and easier to validate

## Common Mistakes

- **Reinventing a Full Language:** Scope creep turns a "simple rules DSL" into an unmaintainable ad hoc programming language
- **No Input Validation:** Feeding untrusted input directly into the interpreter without sanitization or sandboxing
- **Ignoring Precedence/Associativity:** Naive parsers (like the arithmetic example above) silently produce wrong results for real expression languages
- **Interpreting in Hot Paths:** Using an interpreter loop where compiled code is actually required for performance-critical logic

## Related Patterns

- Interpreter Pattern (Gang of Four) — the class-level design pattern this architectural style scales up from
- [Strategy Pattern](../../patterns/strategy-pattern/strategy-pattern.md) — often used to implement individual rule/operator evaluation
- [Domain-Driven Architecture](../domain-driven-architecture/domain-driven-architecture.md) — a DSL is one way to make the ubiquitous language executable
- [Spec-Driven Architecture](../spec-driven-architecture/spec-driven-architecture.md) — acceptance criteria are sometimes expressed in a small interpretable grammar (Gherkin)

## References & Sources

- Gang of Four — "Design Patterns: Elements of Reusable Object-Oriented Software" (1994), Interpreter Pattern
- Robert Nystrom — "Crafting Interpreters" (2021): https://craftinginterpreters.com/
- Martin Fowler — "Domain-Specific Languages" (2010)
