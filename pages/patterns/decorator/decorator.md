# Decorator Pattern

Dynamically adds responsibility to an object by wrapping it, without affecting other instances. Provides a flexible alternative to inheritance.

## Purpose

- Add new features to objects dynamically
- Avoid creating many subclasses
- Compose behaviors at runtime
- Keep single responsibility principle

## Wrapper/Structural Example

```typescript
// Base interface
interface Coffee {
  cost(): number;
  description(): string;
}

// Concrete component
class SimpleCoffee implements Coffee {
  cost() { return 2; }
  description() { return 'Simple Coffee'; }
}

// Decorator: adds responsibility
class MilkDecorator implements Coffee {
  constructor(private coffee: Coffee) { }

  cost() { return this.coffee.cost() + 0.5; }
  description() { return this.coffee.description() + ', Milk'; }
}

class SugarDecorator implements Coffee {
  constructor(private coffee: Coffee) { }

  cost() { return this.coffee.cost() + 0.2; }
  description() { return this.coffee.description() + ', Sugar'; }
}

// Usage: compose decorators
let coffee: Coffee = new SimpleCoffee();
coffee = new MilkDecorator(coffee);
coffee = new SugarDecorator(coffee);

console.log(coffee.description()); // Simple Coffee, Milk, Sugar
console.log(coffee.cost());        // 2.7
```

## TypeScript Decorator Example

```typescript
// Method decorator
function Logger(target: any, key: string, descriptor: PropertyDescriptor) {
  const originalMethod = descriptor.value;

  descriptor.value = function(...args: any[]) {
    console.log(`Calling ${key} with args:`, args);
    const result = originalMethod.apply(this, args);
    console.log(`Result:`, result);
    return result;
  };

  return descriptor;
}

class Calculator {
  @Logger
  add(a: number, b: number) {
    return a + b;
  }
}

const calc = new Calculator();
calc.add(2, 3);
// Calling add with args: [2, 3]
// Result: 5
```

## Parameter Validation Decorator

```typescript
function validateParam(min: number, max: number) {
  return function(target: any, key: string, index: number) {
    const originalMethod = target[key];

    target[key] = function(...args: any[]) {
      const arg = args[index];
      if (arg < min || arg > max) {
        throw new Error(`Argument at index ${index} must be between ${min} and ${max}.`);
      }
      return originalMethod.apply(this, args);
    };
  };
}

class MathOperations {
  @validateParam(0, 10)
  multiply(a: number, b: number) {
    return a * b;
  }
}

const math = new MathOperations();
math.multiply(5, 3);    // Works: 15
math.multiply(5, 12);   // Error: out of range
```

## When to Use

- Need to add features to objects without modifying them
- Have many possible combinations of features
- Want to avoid subclass explosion
- Need runtime composition of behaviors

## Key Differences

| Pattern | Purpose |
|---------|---------|
| Decorator | Add behavior to objects dynamically |
| Adapter | Convert interface to another |
| Facade | Simplify complex subsystem |
| Proxy | Control access to object |
