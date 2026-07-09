# Facade

Provides a unified, simplified interface to a set of interfaces in a subsystem. Hides complexity by providing a single entry point.

## Purpose

- Simplify complex subsystems
- Reduce coupling between clients and complex components
- Provide a convenient, consistent API

## Example: Computer Startup

Without Facade:
```typescript
const cpu = new CPU();
const memory = new Memory();
const drive = new Drive();

cpu.freeze();
memory.load(BOOT_ADDRESS, boot);
drive.read(BOOT_SECTOR, SECTOR_SIZE);
cpu.jump(BOOT_ADDRESS);
cpu.execute();
```

With Facade:
```typescript
class ComputerFacade {
  private cpu = new CPU();
  private memory = new Memory();
  private drive = new Drive();

  startComputer() {
    this.cpu.freeze();
    this.memory.load(BOOT_ADDRESS, this.drive.read(BOOT_SECTOR, SECTOR_SIZE));
    this.cpu.jump(BOOT_ADDRESS);
    this.cpu.execute();
  }
}

const computer = new ComputerFacade();
computer.startComputer(); // Single method call
```

## Example: Payment Processing

```typescript
// Complex subsystems
class PaymentGateway { process() { } }
class FraudDetection { check() { } }
class NotificationService { notify() { } }
class AuditLog { log() { } }

// Facade simplifies the interface
class PaymentFacade {
  private gateway = new PaymentGateway();
  private fraud = new FraudDetection();
  private notification = new NotificationService();
  private audit = new AuditLog();

  processPayment(payment: Payment): boolean {
    if (!this.fraud.check(payment)) {
      return false;
    }

    const result = this.gateway.process(payment);
    this.notification.notify(payment.userId, 'Payment processed');
    this.audit.log(payment);

    return result;
  }
}
```

## Key Characteristics

- **Simplicity:** Reduces learning curve for clients
- **Loose Coupling:** Clients depend on Facade, not subsystem components
- **Optional:** Client can still use subsystem directly if needed
- **Not a Restriction:** Doesn't reduce functionality, just simplifies access

## When to Use

- Complex libraries or frameworks need simplified interface
- Multiple subsystems work together
- Want to layer subsystems hierarchically
- Need to isolate clients from subsystem components
