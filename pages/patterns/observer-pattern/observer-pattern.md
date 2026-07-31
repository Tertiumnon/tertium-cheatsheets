# Observer Pattern

The Observer Pattern defines a one-to-many dependency between objects so that when one object changes state, all its dependents are notified automatically. It's fundamental for reactive programming and event-driven systems.

## Problem

- Need to notify multiple objects about state changes
- Don't want tight coupling between objects
- Need loose coupling so observers can be added/removed dynamically

## Solution

Create a publisher-subscriber mechanism where:
- **Subject (Observable):** Maintains a list of observers and notifies them of state changes
- **Observer:** Defines an interface for receiving notifications
- **Concrete Observer:** Implements the observer interface and reacts to notifications

## Structure

```
┌─────────────┐
│   Subject   │
└──────┬──────┘
       │ notifies
       ├──────────────────┬─────────────────┐
       │                  │                 │
   ┌───▼──────┐      ┌───▼──────┐     ┌───▼──────┐
   │Observer1 │      │Observer2 │     │Observer3 │
   └──────────┘      └──────────┘     └──────────┘
```

## Basic Implementation

### Type 1: Property-Based Observer

```typescript
// Subject
class User {
  private _email: string = '';
  private observers: ((email: string) => void)[] = [];

  get email(): string {
    return this._email;
  }

  set email(value: string) {
    this._email = value;
    this.notifyObservers();
  }

  subscribe(observer: (email: string) => void): void {
    this.observers.push(observer);
  }

  unsubscribe(observer: (email: string) => void): void {
    this.observers = this.observers.filter(obs => obs !== observer);
  }

  private notifyObservers(): void {
    this.observers.forEach(observer => observer(this._email));
  }
}

// Observers
const user = new User();

user.subscribe((email) => {
  console.log(`Email changed to: ${email}`);
});

user.subscribe((email) => {
  console.log(`Sending verification email to: ${email}`);
});

user.email = 'john@example.com';
// Output:
// Email changed to: john@example.com
// Sending verification email to: john@example.com
```

### Type 2: Event-Based Observer

```typescript
// Subject with event emitter
class EventEmitter<T> {
  private observers: Map<string, Set<(data: T) => void>> = new Map();

  on(event: string, observer: (data: T) => void): void {
    if (!this.observers.has(event)) {
      this.observers.set(event, new Set());
    }
    this.observers.get(event)!.add(observer);
  }

  off(event: string, observer: (data: T) => void): void {
    if (this.observers.has(event)) {
      this.observers.get(event)!.delete(observer);
    }
  }

  emit(event: string, data: T): void {
    if (this.observers.has(event)) {
      this.observers.get(event)!.forEach(observer => observer(data));
    }
  }

  once(event: string, observer: (data: T) => void): void {
    const wrapper = (data: T) => {
      observer(data);
      this.off(event, wrapper);
    };
    this.on(event, wrapper);
  }
}

// Usage
interface UserEvent {
  userId: string;
  timestamp: Date;
}

class UserService extends EventEmitter<UserEvent> {
  createUser(userId: string): void {
    // ... create user logic ...
    this.emit('user:created', { userId, timestamp: new Date() });
  }

  updateUser(userId: string): void {
    // ... update user logic ...
    this.emit('user:updated', { userId, timestamp: new Date() });
  }

  deleteUser(userId: string): void {
    // ... delete user logic ...
    this.emit('user:deleted', { userId, timestamp: new Date() });
  }
}

// Observers
const userService = new UserService();

userService.on('user:created', (event) => {
  console.log(`User created: ${event.userId}`);
});

userService.on('user:created', (event) => {
  console.log(`Sending welcome email to user: ${event.userId}`);
});

userService.on('user:updated', (event) => {
  console.log(`User updated: ${event.userId}`);
});

userService.createUser('user123');
```

## Modern Implementation: RxJS Observable

```typescript
import { Subject, Observable } from 'rxjs';
import { filter, map } from 'rxjs/operators';

// Subject (Publisher)
class UserStore {
  private userSubject = new Subject<{ id: string; name: string }>();
  public user$: Observable<{ id: string; name: string }> = this.userSubject.asObservable();

  updateUser(id: string, name: string): void {
    this.userSubject.next({ id, name });
  }

  complete(): void {
    this.userSubject.complete();
  }
}

// Observers
const userStore = new UserStore();

// Observer 1: Log all user changes
userStore.user$.subscribe(user => {
  console.log(`User changed: ${user.name}`);
});

// Observer 2: Send notification for specific users
userStore.user$
  .pipe(
    filter(user => user.id === 'admin'),
    map(user => user.name.toUpperCase())
  )
  .subscribe(name => {
    console.log(`Admin user updated: ${name}`);
  });

userStore.updateUser('user1', 'John Doe');
userStore.updateUser('admin', 'Admin User');
```

## React Hook Observer Pattern

```typescript
// Custom Hook (Observable)
function useUserObservable() {
  const [user, setUser] = useState<User | null>(null);
  const observersRef = useRef<Set<(user: User) => void>>(new Set());

  const subscribe = useCallback((observer: (user: User) => void) => {
    observersRef.current.add(observer);
    return () => observersRef.current.delete(observer);
  }, []);

  const updateUser = useCallback((newUser: User) => {
    setUser(newUser);
    observersRef.current.forEach(observer => observer(newUser));
  }, []);

  return { user, updateUser, subscribe };
}

// Component Observer
function App() {
  const userObservable = useUserObservable();

  useEffect(() => {
    // Subscribe to user changes
    return userObservable.subscribe((user) => {
      console.log('User changed:', user);
    });
  }, []);

  return (
    <button onClick={() => userObservable.updateUser({ id: '1', name: 'Jane' })}>
      Update User
    </button>
  );
}
```

## Real-World Example: E-commerce Order System

```typescript
interface OrderEvent {
  orderId: string;
  status: 'created' | 'confirmed' | 'shipped' | 'delivered';
  timestamp: Date;
}

class Order extends EventEmitter<OrderEvent> {
  private id: string;
  private status: 'created' | 'confirmed' | 'shipped' | 'delivered' = 'created';

  constructor(id: string) {
    super();
    this.id = id;
    this.emit('order:event', {
      orderId: id,
      status: this.status,
      timestamp: new Date()
    });
  }

  confirm(): void {
    this.status = 'confirmed';
    this.emit('order:event', {
      orderId: this.id,
      status: this.status,
      timestamp: new Date()
    });
  }

  ship(): void {
    this.status = 'shipped';
    this.emit('order:event', {
      orderId: this.id,
      status: this.status,
      timestamp: new Date()
    });
  }

  deliver(): void {
    this.status = 'delivered';
    this.emit('order:event', {
      orderId: this.id,
      status: this.status,
      timestamp: new Date()
    });
  }
}

// Observers
class EmailNotifier {
  constructor(order: Order) {
    order.on('order:event', (event) => {
      switch (event.status) {
        case 'confirmed':
          console.log(`Sending confirmation email for order ${event.orderId}`);
          break;
        case 'shipped':
          console.log(`Sending shipping notification for order ${event.orderId}`);
          break;
        case 'delivered':
          console.log(`Sending delivery confirmation for order ${event.orderId}`);
          break;
      }
    });
  }
}

class OrderLogger {
  constructor(order: Order) {
    order.on('order:event', (event) => {
      console.log(`[LOG] Order ${event.orderId} status changed to ${event.status}`);
    });
  }
}

class InventoryManager {
  constructor(order: Order) {
    order.on('order:event', (event) => {
      if (event.status === 'confirmed') {
        console.log(`[INVENTORY] Reserving items for order ${event.orderId}`);
      }
      if (event.status === 'shipped') {
        console.log(`[INVENTORY] Removing items from stock for order ${event.orderId}`);
      }
    });
  }
}

// Usage
const order = new Order('ORD-001');
new EmailNotifier(order);
new OrderLogger(order);
new InventoryManager(order);

order.confirm();  // Triggers all observers
order.ship();     // Triggers all observers
order.deliver();  // Triggers all observers
```

## Advantages

- **Loose Coupling:** Subject and observers are decoupled
- **Dynamic Relationships:** Observers can be added/removed at runtime
- **Support for Broadcast:** Multiple observers can react to the same event
- **Reactive Programming:** Natural fit for reactive systems
- **Separation of Concerns:** Each observer handles one responsibility

## Disadvantages

- **Complexity:** Can become complex with many observers
- **Performance:** Many observers can impact performance
- **Memory Leaks:** Observers must be properly unsubscribed
- **Unpredictable Order:** Observer execution order is unpredictable
- **Debugging:** Hard to trace which observer is being called

## Best Practices

- **Memory Management:** Always unsubscribe from observers
- **Error Handling:** Handle errors in observer callbacks
- **Avoid Circular Dependencies:** Don't make observers modify the subject
- **Single Responsibility:** Each observer should do one thing
- **Use TypeScript:** Type the events for safety
- **Naming Conventions:** Use clear event names (e.g., `user:created`, `order:shipped`)
- **Documentation:** Document what events are emitted

## Related Patterns

- **[Pub/Sub Pattern](../pub-sub-pattern/pub-sub-pattern.md):** Similar but usually decoupled via a message broker instead of direct references
- **Event Emitter:** Implementation pattern for observers
- **Reactive Extensions (RxJS):** Modern implementation with additional operators

## References & Sources

- Gang of Four - "Design Patterns" (Observer Pattern)
- RxJS: https://rxjs.dev/
- Node.js EventEmitter: https://nodejs.org/api/events.html
- Observer Pattern: https://refactoring.guru/design-patterns/observer
