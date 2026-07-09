# Builder Pattern

The Builder Pattern is a creational pattern that separates the construction of a complex object from its representation, allowing the same construction process to create different representations. It's useful for objects with many optional parameters.

## Problem

- Object has many optional parameters
- Constructor parameters are hard to remember
- Need multiple constructor overloads
- Want to construct complex objects step-by-step
- Need readable and flexible object construction

## Solution

Create a builder class that constructs objects step-by-step, returning a configured object at the end.

## Structure

```
┌──────────────┐
│   Builder    │
├──────────────┤
│ -property1   │
│ -property2   │
│ build()      │
└──────┬───────┘
       │ creates
       ▼
┌──────────────┐
│   Complex    │
│   Object     │
└──────────────┘
```

## Basic Implementation

### Simple Builder

```typescript
// Complex Object
class User {
  id: string;
  name: string;
  email: string;
  age?: number;
  role?: string;
  avatar?: string;
  isActive?: boolean;

  constructor(
    id: string,
    name: string,
    email: string,
    age?: number,
    role?: string,
    avatar?: string,
    isActive?: boolean
  ) {
    this.id = id;
    this.name = name;
    this.email = email;
    this.age = age;
    this.role = role;
    this.avatar = avatar;
    this.isActive = isActive;
  }
}

// Builder Class
class UserBuilder {
  private id: string = '';
  private name: string = '';
  private email: string = '';
  private age?: number;
  private role?: string;
  private avatar?: string;
  private isActive: boolean = true;

  setId(id: string): this {
    this.id = id;
    return this;
  }

  setName(name: string): this {
    this.name = name;
    return this;
  }

  setEmail(email: string): this {
    this.email = email;
    return this;
  }

  setAge(age: number): this {
    this.age = age;
    return this;
  }

  setRole(role: string): this {
    this.role = role;
    return this;
  }

  setAvatar(avatar: string): this {
    this.avatar = avatar;
    return this;
  }

  setActive(isActive: boolean): this {
    this.isActive = isActive;
    return this;
  }

  build(): User {
    if (!this.id || !this.name || !this.email) {
      throw new Error('id, name, and email are required');
    }
    return new User(
      this.id,
      this.name,
      this.email,
      this.age,
      this.role,
      this.avatar,
      this.isActive
    );
  }
}

// Usage
const user = new UserBuilder()
  .setId('1')
  .setName('John Doe')
  .setEmail('john@example.com')
  .setAge(30)
  .setRole('admin')
  .setAvatar('https://example.com/avatar.jpg')
  .build();

console.log(user);
```

## Fluent Builder Pattern

```typescript
// Configuration object
interface DatabaseConfig {
  host: string;
  port: number;
  database: string;
  username: string;
  password: string;
  ssl?: boolean;
  timeout?: number;
  maxConnections?: number;
  retryAttempts?: number;
}

// Database Builder
class DatabaseConfigBuilder {
  private config: Partial<DatabaseConfig> = {};

  constructor(host: string, port: number, database: string) {
    this.config.host = host;
    this.config.port = port;
    this.config.database = database;
  }

  withCredentials(username: string, password: string): this {
    this.config.username = username;
    this.config.password = password;
    return this;
  }

  withSSL(enabled: boolean = true): this {
    this.config.ssl = enabled;
    return this;
  }

  withTimeout(ms: number): this {
    this.config.timeout = ms;
    return this;
  }

  withMaxConnections(max: number): this {
    this.config.maxConnections = max;
    return this;
  }

  withRetry(attempts: number): this {
    this.config.retryAttempts = attempts;
    return this;
  }

  build(): DatabaseConfig {
    const required = ['host', 'port', 'database', 'username', 'password'];
    for (const field of required) {
      if (!(field in this.config)) {
        throw new Error(`Missing required field: ${field}`);
      }
    }
    return this.config as DatabaseConfig;
  }
}

// Usage
const dbConfig = new DatabaseConfigBuilder('localhost', 5432, 'myapp')
  .withCredentials('admin', 'password123')
  .withSSL(true)
  .withTimeout(5000)
  .withMaxConnections(20)
  .withRetry(3)
  .build();

console.log(dbConfig);
```

## Builder with Director Pattern

```typescript
// Product
interface House {
  foundation: string;
  walls: string;
  roof: string;
  windows: number;
  doors: number;
  garage: boolean;
}

// Builder Interface
interface HouseBuilder {
  buildFoundation(): void;
  buildWalls(): void;
  buildRoof(): void;
  buildWindows(): void;
  buildDoors(): void;
  buildGarage(): void;
  getHouse(): House;
}

// Concrete Builders
class WoodHouseBuilder implements HouseBuilder {
  private house: House = {
    foundation: '',
    walls: '',
    roof: '',
    windows: 0,
    doors: 0,
    garage: false
  };

  buildFoundation(): void {
    this.house.foundation = 'Concrete foundation';
  }

  buildWalls(): void {
    this.house.walls = 'Wood walls';
  }

  buildRoof(): void {
    this.house.roof = 'Wood roof';
  }

  buildWindows(): void {
    this.house.windows = 6;
  }

  buildDoors(): void {
    this.house.doors = 2;
  }

  buildGarage(): void {
    this.house.garage = true;
  }

  getHouse(): House {
    return this.house;
  }
}

class GlassHouseBuilder implements HouseBuilder {
  private house: House = {
    foundation: '',
    walls: '',
    roof: '',
    windows: 0,
    doors: 0,
    garage: false
  };

  buildFoundation(): void {
    this.house.foundation = 'Steel foundation';
  }

  buildWalls(): void {
    this.house.walls = 'Glass walls';
  }

  buildRoof(): void {
    this.house.roof = 'Glass roof';
  }

  buildWindows(): void {
    this.house.windows = 12;
  }

  buildDoors(): void {
    this.house.doors = 3;
  }

  buildGarage(): void {
    this.house.garage = false;
  }

  getHouse(): House {
    return this.house;
  }
}

// Director
class HouseDirector {
  constructor(private builder: HouseBuilder) {}

  buildSimpleHouse(): void {
    this.builder.buildFoundation();
    this.builder.buildWalls();
    this.builder.buildRoof();
    this.builder.buildDoors();
  }

  buildLuxuryHouse(): void {
    this.builder.buildFoundation();
    this.builder.buildWalls();
    this.builder.buildRoof();
    this.builder.buildWindows();
    this.builder.buildDoors();
    this.builder.buildGarage();
  }

  getHouse(): House {
    return this.builder.getHouse();
  }
}

// Usage
const woodBuilder = new WoodHouseBuilder();
const director = new HouseDirector(woodBuilder);

director.buildLuxuryHouse();
const luxuryWoodHouse = director.getHouse();

const glassBuilder = new GlassHouseBuilder();
director = new HouseDirector(glassBuilder);
director.buildSimpleHouse();
const simpleGlassHouse = director.getHouse();
```

## Real-World Example: HTTP Request Builder

```typescript
interface HttpRequest {
  method: 'GET' | 'POST' | 'PUT' | 'DELETE';
  url: string;
  headers: Record<string, string>;
  body?: any;
  timeout?: number;
  retries?: number;
}

class HttpRequestBuilder {
  private method: 'GET' | 'POST' | 'PUT' | 'DELETE' = 'GET';
  private url: string = '';
  private headers: Record<string, string> = {};
  private body?: any;
  private timeout: number = 5000;
  private retries: number = 1;

  setMethod(method: 'GET' | 'POST' | 'PUT' | 'DELETE'): this {
    this.method = method;
    return this;
  }

  setUrl(url: string): this {
    this.url = url;
    return this;
  }

  addHeader(key: string, value: string): this {
    this.headers[key] = value;
    return this;
  }

  setBody(body: any): this {
    this.body = body;
    return this;
  }

  setTimeout(ms: number): this {
    this.timeout = ms;
    return this;
  }

  setRetries(count: number): this {
    this.retries = count;
    return this;
  }

  withAuth(token: string): this {
    this.headers['Authorization'] = `Bearer ${token}`;
    return this;
  }

  withJsonBody(data: any): this {
    this.headers['Content-Type'] = 'application/json';
    this.body = JSON.stringify(data);
    return this;
  }

  build(): HttpRequest {
    if (!this.url) {
      throw new Error('URL is required');
    }
    return {
      method: this.method,
      url: this.url,
      headers: this.headers,
      body: this.body,
      timeout: this.timeout,
      retries: this.retries
    };
  }
}

// Usage
const request = new HttpRequestBuilder()
  .setMethod('POST')
  .setUrl('https://api.example.com/users')
  .withAuth('my-secret-token')
  .withJsonBody({ name: 'John', email: 'john@example.com' })
  .setTimeout(10000)
  .setRetries(3)
  .build();

console.log(request);
```

## SQL Query Builder Example

```typescript
interface QueryOptions {
  select: string[];
  from: string;
  where: Array<{ column: string; operator: string; value: any }>;
  orderBy: Array<{ column: string; direction: 'ASC' | 'DESC' }>;
  limit?: number;
  offset?: number;
}

class QueryBuilder {
  private select: string[] = ['*'];
  private from: string = '';
  private where: Array<{ column: string; operator: string; value: any }> = [];
  private orderBy: Array<{ column: string; direction: 'ASC' | 'DESC' }> = [];
  private limit?: number;
  private offset?: number;

  select(...columns: string[]): this {
    this.select = columns.length > 0 ? columns : ['*'];
    return this;
  }

  from(table: string): this {
    this.from = table;
    return this;
  }

  where(column: string, operator: string, value: any): this {
    this.where.push({ column, operator, value });
    return this;
  }

  orderBy(column: string, direction: 'ASC' | 'DESC' = 'ASC'): this {
    this.orderBy.push({ column, direction });
    return this;
  }

  limit(limit: number): this {
    this.limit = limit;
    return this;
  }

  offset(offset: number): this {
    this.offset = offset;
    return this;
  }

  build(): string {
    if (!this.from) throw new Error('FROM clause is required');

    let query = `SELECT ${this.select.join(', ')} FROM ${this.from}`;

    if (this.where.length > 0) {
      const conditions = this.where.map(
        w => `${w.column} ${w.operator} '${w.value}'`
      ).join(' AND ');
      query += ` WHERE ${conditions}`;
    }

    if (this.orderBy.length > 0) {
      const orders = this.orderBy.map(
        o => `${o.column} ${o.direction}`
      ).join(', ');
      query += ` ORDER BY ${orders}`;
    }

    if (this.limit) query += ` LIMIT ${this.limit}`;
    if (this.offset) query += ` OFFSET ${this.offset}`;

    return query;
  }
}

// Usage
const query = new QueryBuilder()
  .select('id', 'name', 'email')
  .from('users')
  .where('age', '>', 18)
  .where('status', '=', 'active')
  .orderBy('name', 'ASC')
  .limit(10)
  .offset(0)
  .build();

console.log(query);
// SELECT id, name, email FROM users WHERE age > '18' AND status = 'active' ORDER BY name ASC LIMIT 10 OFFSET 0
```

## Advantages

- **Readable:** Clear, fluent API for object construction
- **Flexible:** Easy to add optional parameters
- **Encapsulation:** Hides construction complexity
- **Validation:** Can validate before returning object
- **Step-by-Step:** Build objects incrementally
- **Immutable:** Can create immutable objects

## Disadvantages

- **Boilerplate:** More code for simple objects
- **Memory Overhead:** Builder instance uses additional memory
- **Complexity:** Overkill for objects with few parameters

## When to Use

- Objects with many optional parameters
- Complex object construction
- Want readable, fluent API
- Need to validate before creating object
- Objects have interdependent parameters

## Best Practices

- **Immutable Results:** Create immutable final objects
- **Validation:** Validate required fields in `build()`
- **Default Values:** Set reasonable defaults
- **Fluent Interface:** Return `this` for method chaining
- **Clear Naming:** Use descriptive method names
- **Type Safety:** Use TypeScript for compile-time checking
- **Documentation:** Document builder patterns

## Related Patterns

- **Abstract Factory:** Alternative for object creation
- **Factory Method:** Simpler alternative for less complex objects
- **Director Pattern:** Use directors to orchestrate complex builds

## References & Sources

- Gang of Four - "Design Patterns" (Builder Pattern)
- Refactoring.Guru - Builder Pattern: https://refactoring.guru/design-patterns/builder
- Joshua Bloch - Effective Java (Builder Pattern)
