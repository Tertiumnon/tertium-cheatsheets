# Adapter

Converts one interface to another so that it matches what the client is expecting. Allows incompatible interfaces to work together.

## Purpose

- Adapt legacy code to new interfaces
- Convert third-party library interfaces to your own
- Make incompatible classes work together

## Class Adapter Example

```typescript
// Old incompatible interface
class OldPaymentSystem {
  collectPayment(id: string, amount: number) {
    console.log(`Old system: Collected ${amount} from ${id}`);
  }
}

// New expected interface
interface PaymentProcessor {
  process(customerId: string, price: number): void;
}

// Adapter bridges old and new
class OldPaymentAdapter implements PaymentProcessor {
  constructor(private oldSystem: OldPaymentSystem) { }

  process(customerId: string, price: number) {
    this.oldSystem.collectPayment(customerId, price);
  }
}

// Client uses new interface
const adapter = new OldPaymentAdapter(new OldPaymentSystem());
adapter.process('cust_123', 99.99);
```

## Real-World Example: API Integration

```typescript
// Third-party API returns different format
class ThirdPartyWeatherAPI {
  getWeather() {
    return {
      temp_celsius: 25,
      humidity_percent: 65,
      wind_kmh: 10
    };
  }
}

// Our app expects this interface
interface Weather {
  temperature: number;
  humidity: number;
  windSpeed: number;
}

// Adapter converts formats
class WeatherAdapter implements Weather {
  private api = new ThirdPartyWeatherAPI();
  private data = this.api.getWeather();

  get temperature(): number {
    return this.data.temp_celsius;
  }

  get humidity(): number {
    return this.data.humidity_percent;
  }

  get windSpeed(): number {
    return this.data.wind_kmh;
  }
}

const weather = new WeatherAdapter();
console.log(weather.temperature); // Uses adapted interface
```

## Object Adapter Example

```typescript
// Different list interfaces
class JavaList {
  add(item: any) { }
  remove(item: any) { }
  size(): number { return 0; }
}

interface JavaScriptArray {
  push(item: any): void;
  pop(): any;
  length: number;
}

// Adapter bridges them
class ListToArrayAdapter implements JavaScriptArray {
  constructor(private list: JavaList) { }

  push(item: any) {
    this.list.add(item);
  }

  pop() {
    // Simulate pop with remove
    const size = this.list.size();
    return size > 0 ? true : false;
  }

  get length(): number {
    return this.list.size();
  }
}
```

## When to Use

- Working with legacy code that can't be modified
- Integrating third-party libraries with incompatible interfaces
- Creating wrapper classes for external APIs
- Allowing classes with incompatible interfaces to collaborate

## Variants

- **Class Adapter:** Uses inheritance (Java-style)
- **Object Adapter:** Uses composition (more flexible, preferred)
