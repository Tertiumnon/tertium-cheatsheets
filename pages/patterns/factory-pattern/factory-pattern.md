# Factory Pattern

The Factory Pattern is a creational pattern that provides an interface for creating objects without specifying the exact classes of objects to create. It encapsulates object creation logic, making code more flexible and maintainable.

## Problem

- Need to create objects of different types based on conditions
- Object creation logic is complex
- Want to avoid tight coupling to specific classes
- Need flexibility to change object creation without affecting client code

## Solution

Create a factory (method or class) that handles object creation, returning objects through a common interface.

## Structure

```
      ┌─────────────┐
      │   Client    │
      └──────┬──────┘
             │ uses
      ┌──────▼──────────┐
      │ ObjectFactory   │
      │ createObject()  │
      └──────┬──────────┘
             │
      ┌──────┴──────────┐
      │   creates       │
      │                 │
   ┌──▼───┐   ┌────▼───┐   ┌──────┐
   │TypeA │   │TypeB   │   │TypeC │
   └──────┘   └────────┘   └──────┘
```

## Simple Factory Pattern

```typescript
// Product Interface
interface Database {
  connect(): void;
  query(sql: string): Promise<any>;
  disconnect(): void;
}

// Concrete Products
class PostgresDatabase implements Database {
  connect(): void {
    console.log('Connected to PostgreSQL');
  }

  async query(sql: string): Promise<any> {
    console.log(`Executing PostgreSQL query: ${sql}`);
    return { rows: [] };
  }

  disconnect(): void {
    console.log('Disconnected from PostgreSQL');
  }
}

class MySQLDatabase implements Database {
  connect(): void {
    console.log('Connected to MySQL');
  }

  async query(sql: string): Promise<any> {
    console.log(`Executing MySQL query: ${sql}`);
    return { rows: [] };
  }

  disconnect(): void {
    console.log('Disconnected from MySQL');
  }
}

class MongoDBDatabase implements Database {
  connect(): void {
    console.log('Connected to MongoDB');
  }

  async query(sql: string): Promise<any> {
    console.log(`Executing MongoDB query: ${sql}`);
    return { documents: [] };
  }

  disconnect(): void {
    console.log('Disconnected from MongoDB');
  }
}

// Factory Class
class DatabaseFactory {
  static createDatabase(type: 'postgres' | 'mysql' | 'mongodb'): Database {
    switch (type) {
      case 'postgres':
        return new PostgresDatabase();
      case 'mysql':
        return new MySQLDatabase();
      case 'mongodb':
        return new MongoDBDatabase();
      default:
        throw new Error(`Unknown database type: ${type}`);
    }
  }
}

// Usage
const dbType = process.env.DB_TYPE || 'postgres';
const database = DatabaseFactory.createDatabase(dbType as any);
database.connect();
database.query('SELECT * FROM users');
database.disconnect();
```

## Factory Method Pattern

```typescript
// Product Interface
interface Logger {
  log(message: string): void;
}

// Concrete Products
class ConsoleLogger implements Logger {
  log(message: string): void {
    console.log(`[CONSOLE] ${message}`);
  }
}

class FileLogger implements Logger {
  constructor(private filePath: string) {}

  log(message: string): void {
    console.log(`[FILE] Writing to ${this.filePath}: ${message}`);
    // Write to file
  }
}

class CloudLogger implements Logger {
  constructor(private endpoint: string) {}

  log(message: string): void {
    console.log(`[CLOUD] Sending to ${this.endpoint}: ${message}`);
    // Send to cloud service
  }
}

// Creator Interface
abstract class LoggerCreator {
  abstract createLogger(): Logger;

  log(message: string): void {
    const logger = this.createLogger();
    logger.log(message);
  }
}

// Concrete Creators
class ConsoleLoggerCreator extends LoggerCreator {
  createLogger(): Logger {
    return new ConsoleLogger();
  }
}

class FileLoggerCreator extends LoggerCreator {
  private filePath: string;

  constructor(filePath: string) {
    super();
    this.filePath = filePath;
  }

  createLogger(): Logger {
    return new FileLogger(this.filePath);
  }
}

class CloudLoggerCreator extends LoggerCreator {
  private endpoint: string;

  constructor(endpoint: string) {
    super();
    this.endpoint = endpoint;
  }

  createLogger(): Logger {
    return new CloudLogger(this.endpoint);
  }
}

// Usage
const creators: LoggerCreator[] = [
  new ConsoleLoggerCreator(),
  new FileLoggerCreator('/var/log/app.log'),
  new CloudLoggerCreator('https://logs.example.com')
];

creators.forEach(creator => {
  creator.log('Application started');
});
```

## Abstract Factory Pattern

```typescript
// Product Interfaces
interface Button {
  render(): string;
}

interface TextField {
  render(): string;
}

interface UIFactory {
  createButton(): Button;
  createTextField(): TextField;
}

// Concrete Products - Dark Theme
class DarkButton implements Button {
  render(): string {
    return '<button style="background: #333; color: white">Dark Button</button>';
  }
}

class DarkTextField implements TextField {
  render(): string {
    return '<input style="background: #333; color: white" />';
  }
}

// Concrete Products - Light Theme
class LightButton implements Button {
  render(): string {
    return '<button style="background: white; color: black">Light Button</button>';
  }
}

class LightTextField implements TextField {
  render(): string {
    return '<input style="background: white; color: black" />';
  }
}

// Concrete Factories
class DarkThemeFactory implements UIFactory {
  createButton(): Button {
    return new DarkButton();
  }

  createTextField(): TextField {
    return new DarkTextField();
  }
}

class LightThemeFactory implements UIFactory {
  createButton(): Button {
    return new LightButton();
  }

  createTextField(): TextField {
    return new LightTextField();
  }
}

// Application
class Application {
  private button: Button;
  private textField: TextField;

  constructor(factory: UIFactory) {
    this.button = factory.createButton();
    this.textField = factory.createTextField();
  }

  render(): string {
    return `
      <div>
        ${this.button.render()}
        ${this.textField.render()}
      </div>
    `;
  }
}

// Usage
const theme = 'dark';
const factory = theme === 'dark' ? new DarkThemeFactory() : new LightThemeFactory();
const app = new Application(factory);
console.log(app.render());
```

## Real-World Example: Transport Factory

```typescript
interface Transport {
  deliver(cargo: string, destination: string): void;
}

class Truck implements Transport {
  deliver(cargo: string, destination: string): void {
    console.log(`Truck delivering ${cargo} to ${destination}`);
  }
}

class Plane implements Transport {
  deliver(cargo: string, destination: string): void {
    console.log(`Plane delivering ${cargo} to ${destination}`);
  }
}

class Ship implements Transport {
  deliver(cargo: string, destination: string): void {
    console.log(`Ship delivering ${cargo} to ${destination}`);
  }
}

// Factory with configuration
class TransportFactory {
  static createTransport(
    distance: number,
    weight: number,
    cost: 'cheap' | 'balanced' | 'fast'
  ): Transport {
    if (distance > 1000 && cost !== 'cheap') {
      return new Plane();
    }
    if (weight > 10000) {
      return new Ship();
    }
    return new Truck();
  }
}

class LogisticsService {
  ship(cargo: string, destination: string, distance: number, weight: number): void {
    const transport = TransportFactory.createTransport(distance, weight, 'balanced');
    transport.deliver(cargo, destination);
  }
}

// Usage
const logistics = new LogisticsService();
logistics.ship('Electronics', 'Tokyo', 5000, 500);  // Plane
logistics.ship('Steel', 'Port', 100, 20000);        // Ship
logistics.ship('Packages', 'Downtown', 50, 100);    // Truck
```

## Singleton Factory Pattern

```typescript
class DatabaseConnection {
  private static instance: DatabaseConnection | null = null;
  private connected: boolean = false;

  private constructor() {
    console.log('Creating database connection');
  }

  static getInstance(): DatabaseConnection {
    if (!DatabaseConnection.instance) {
      DatabaseConnection.instance = new DatabaseConnection();
    }
    return DatabaseConnection.instance;
  }

  connect(): void {
    this.connected = true;
    console.log('Connected to database');
  }

  disconnect(): void {
    this.connected = false;
    console.log('Disconnected from database');
  }

  isConnected(): boolean {
    return this.connected;
  }
}

// Usage
const db1 = DatabaseConnection.getInstance();
db1.connect();

const db2 = DatabaseConnection.getInstance();
console.log(db1 === db2); // true
console.log(db2.isConnected()); // true
```

## Factory with Dependency Injection

```typescript
interface DatabaseFactory {
  createConnection(): Promise<void>;
}

class DatabaseService implements DatabaseFactory {
  async createConnection(): Promise<void> {
    console.log('Creating database connection');
  }
}

class ServiceLocator {
  private services: Map<string, any> = new Map();

  register(name: string, factory: () => any): void {
    this.services.set(name, factory);
  }

  get<T>(name: string): T {
    const factory = this.services.get(name);
    if (!factory) {
      throw new Error(`Service ${name} not found`);
    }
    return factory();
  }
}

// Usage
const locator = new ServiceLocator();

locator.register('database', () => new DatabaseService());

const dbService = locator.get<DatabaseService>('database');
dbService.createConnection();
```

## Advantages

- **Encapsulation:** Object creation logic is encapsulated
- **Flexibility:** Easy to add new types without changing client code
- **Loose Coupling:** Client doesn't depend on concrete classes
- **Single Responsibility:** Creation logic is separated
- **Open/Closed Principle:** Open for extension, closed for modification

## Disadvantages

- **Complexity:** Extra classes and interfaces
- **Overhead:** More code for simple cases
- **Indirection:** Adds a layer of indirection

## When to Use

- Multiple similar objects to create
- Object creation is complex
- Object type determined at runtime
- Want to reduce client's coupling to concrete classes

## Best Practices

- **Use Configuration:** Factory parameters from config
- **Error Handling:** Validate parameters and throw meaningful errors
- **Naming:** Name factories clearly (e.g., `createDatabase`, `createLogger`)
- **Type Safety:** Use TypeScript for compile-time checking
- **Documentation:** Document what objects each factory creates

## Related Patterns

- **Singleton Pattern:** Factory can return singleton instances
- **Builder Pattern:** Alternative for complex object creation
- **Abstract Factory:** Factories for related objects
- **Prototype Pattern:** Copy existing objects instead of creating new

## References & Sources

- Gang of Four - "Design Patterns" (Factory Patterns)
- Refactoring.Guru - Factory Method: https://refactoring.guru/design-patterns/factory-method
- Refactoring.Guru - Abstract Factory: https://refactoring.guru/design-patterns/abstract-factory
