# SOLID

Five principles for writing maintainable, scalable code.

- Single Responsibility (SRP)
- Open-Closed (OCP)
- Liskov Substitution (LSP)
- Interface Segregation (ISP)
- Dependency Inversion (DIP)

## Single Responsibility (SRP)

A module should be responsible to one, and only one, actor (reason to change).

**Bad:**
```typescript
class User {
  create() { }
  save() { }
  sendEmail() { }        // Email responsibility
  generateReport() { }   // Report responsibility
}
```

**Good:**
```typescript
class User {
  create() { }
  save() { }
}

class EmailService {
  sendEmail(user) { }
}

class ReportGenerator {
  generateReport(user) { }
}
```

## Open-Closed (OCP)

Software entities should be **open for extension** but **closed for modification**.

**Bad:**
```typescript
class PaymentProcessor {
  process(payment) {
    if (payment.type === 'credit') { /* process */ }
    if (payment.type === 'paypal') { /* process */ }
    if (payment.type === 'bitcoin') { /* process */ }  // Modify for new type
  }
}
```

**Good:**
```typescript
interface PaymentProcessor {
  process(amount: number): void;
}

class CreditCardProcessor implements PaymentProcessor {
  process(amount) { /* credit logic */ }
}

class PayPalProcessor implements PaymentProcessor {
  process(amount) { /* paypal logic */ }
}

// New payment type? Just add new class, don't modify existing
```

## Liskov Substitution (LSP)

Subtypes must be substitutable for their base types without breaking functionality.

**Bad:**
```typescript
class Bird {
  fly() { console.log('Flying'); }
}

class Penguin extends Bird {
  fly() { throw new Error('Penguins cannot fly'); }  // Violates LSP
}
```

**Good:**
```typescript
class Bird { }

class FlyingBird extends Bird {
  fly() { console.log('Flying'); }
}

class Penguin extends Bird {
  swim() { console.log('Swimming'); }
}
```

## Interface Segregation (ISP)

No client should be forced to depend on methods it does not use.

**Bad:**
```typescript
interface Worker {
  work(): void;
  eat(): void;      // Not all workers need to eat
  manage(): void;   // Robots don't need to manage
}

class Robot implements Worker {
  work() { }
  eat() { throw new Error('Robots don\'t eat'); }
  manage() { throw new Error('Robots don\'t manage'); }
}
```

**Good:**
```typescript
interface Workable {
  work(): void;
}

interface Eatable {
  eat(): void;
}

interface Manageable {
  manage(): void;
}

class Robot implements Workable {
  work() { }
}

class Human implements Workable, Eatable, Manageable {
  work() { }
  eat() { }
  manage() { }
}
```

## Dependency Inversion (DIP)

High-level modules should not depend on low-level modules. Both should depend on abstractions.

**Bad:**
```typescript
class UserService {
  private database = new MySQLDatabase();  // Direct dependency on MySQL

  getUser(id) {
    return this.database.query(`SELECT * FROM users WHERE id = ${id}`);
  }
}
```

**Good:**
```typescript
interface Database {
  query(sql: string): any;
}

class UserService {
  constructor(private database: Database) { }

  getUser(id) {
    return this.database.query(`SELECT * FROM users WHERE id = ${id}`);
  }
}

// Can inject any database implementation
const service = new UserService(new MySQLDatabase());
// OR
const service = new UserService(new PostgresDatabase());
```
