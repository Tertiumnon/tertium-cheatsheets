# Object-Oriented Programming (OOP)

Four core pillars of OOP: Abstraction, Encapsulation, Polymorphism, and Inheritance.

Related: 5 principles of OOP are **SOLID**.

## Abstraction

Hide internal complexity and show only essential features.

```typescript
// Abstraction: Users don't need to know HOW payment is processed
class PaymentGateway {
  private apiKey: string;
  private validateCard() { }
  private chargeCard() { }

  // Only expose what's needed
  public processPayment(amount: number) {
    this.validateCard();
    this.chargeCard();
  }
}

const gateway = new PaymentGateway();
gateway.processPayment(100); // Simple interface
```

## Encapsulation

Bundle data and methods that operate on that data into a single unit (class), controlling access levels.

```typescript
class BankAccount {
  private balance = 0;  // Private data
  private pin = '1234';

  // Controlled access through methods
  public deposit(amount: number) {
    if (amount > 0) this.balance += amount;
  }

  public withdraw(amount: number, enteredPin: string) {
    if (enteredPin === this.pin && amount <= this.balance) {
      this.balance -= amount;
    }
  }

  public getBalance(enteredPin: string) {
    return enteredPin === this.pin ? this.balance : null;
  }
}

const account = new BankAccount();
account.deposit(1000);
// account.balance = 500; // Error: Cannot access private member
```

## Polymorphism

Same interface, different implementations. "Many forms."

```typescript
// Polymorphism: Different animals speak differently
interface Animal {
  speak(): void;
}

class Dog implements Animal {
  speak() { console.log('Woof'); }
}

class Cat implements Animal {
  speak() { console.log('Meow'); }
}

class Bird implements Animal {
  speak() { console.log('Tweet'); }
}

function animalSound(animal: Animal) {
  animal.speak(); // Works with any Animal type
}

animalSound(new Dog());   // Woof
animalSound(new Cat());   // Meow
animalSound(new Bird());  // Tweet
```

## Inheritance

Child class derives from parent class, reusing code and extending functionality.

```typescript
// Base class
class Vehicle {
  brand: string;

  constructor(brand: string) {
    this.brand = brand;
  }

  start() {
    console.log(`${this.brand} is starting`);
  }
}

// Derived class
class Car extends Vehicle {
  doors: number;

  constructor(brand: string, doors: number) {
    super(brand);  // Call parent constructor
    this.doors = doors;
  }

  start() {
    super.start(); // Call parent method
    console.log('Car is ready to drive');
  }
}

const car = new Car('Toyota', 4);
car.start();
// Toyota is starting
// Car is ready to drive
```
