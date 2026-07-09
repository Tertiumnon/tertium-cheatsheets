# Strategy Pattern

The Strategy Pattern defines a family of algorithms, encapsulates each one, and makes them interchangeable. It lets the algorithm vary independently from clients that use it. This pattern is useful for selecting an algorithm at runtime.

## Problem

- Multiple algorithms for a task
- Need to switch algorithms dynamically
- Want to avoid conditional statements (if/switch)
- Need to add new algorithms without modifying existing code

## Solution

Encapsulate each algorithm in a separate class implementing a common interface, allowing them to be interchangeable.

## Structure

```
┌──────────────┐
│   Context    │
├──────────────┤
│ strategy     │─────┐
│ execute()    │     │
└──────────────┘     │
                     │
            ┌────────▼────────┐
            │   Strategy      │
            │  (Interface)    │
            └────────┬────────┘
                     │
         ┌───────────┼───────────┐
         │           │           │
     ┌───▼───┐  ┌───▼───┐  ┌───▼───┐
     │StratA │  │StratB │  │StratC │
     └───────┘  └───────┘  └───────┘
```

## Basic Implementation

### Payment Strategy Example

```typescript
// Strategy Interface
interface PaymentStrategy {
  pay(amount: number): boolean;
  refund(amount: number): boolean;
}

// Concrete Strategies
class CreditCardPayment implements PaymentStrategy {
  constructor(
    private cardNumber: string,
    private cvv: string,
    private expiryDate: string
  ) {}

  pay(amount: number): boolean {
    console.log(`Processing credit card payment of $${amount}`);
    // Validate card and process
    return true;
  }

  refund(amount: number): boolean {
    console.log(`Refunding $${amount} to credit card`);
    return true;
  }
}

class PayPalPayment implements PaymentStrategy {
  constructor(private email: string, private password: string) {}

  pay(amount: number): boolean {
    console.log(`Processing PayPal payment of $${amount} from ${this.email}`);
    // PayPal API call
    return true;
  }

  refund(amount: number): boolean {
    console.log(`Refunding $${amount} via PayPal`);
    return true;
  }
}

class CryptoCurrencyPayment implements PaymentStrategy {
  constructor(private walletAddress: string, private cryptoType: string) {}

  pay(amount: number): boolean {
    console.log(`Processing ${this.cryptoType} payment of ${amount} coins`);
    // Blockchain transaction
    return true;
  }

  refund(amount: number): boolean {
    console.log(`Refunding ${amount} ${this.cryptoType}`);
    return true;
  }
}

// Context
class ShoppingCart {
  private items: Array<{ price: number; name: string }> = [];
  private paymentStrategy: PaymentStrategy | null = null;

  addItem(name: string, price: number): void {
    this.items.push({ name, price });
  }

  setPaymentStrategy(strategy: PaymentStrategy): void {
    this.paymentStrategy = strategy;
  }

  checkout(): boolean {
    if (!this.paymentStrategy) {
      throw new Error('Payment strategy not set');
    }

    const total = this.items.reduce((sum, item) => sum + item.price, 0);
    return this.paymentStrategy.pay(total);
  }

  requestRefund(): boolean {
    if (!this.paymentStrategy) {
      throw new Error('Payment strategy not set');
    }

    const total = this.items.reduce((sum, item) => sum + item.price, 0);
    return this.paymentStrategy.refund(total);
  }
}

// Usage
const cart = new ShoppingCart();
cart.addItem('Laptop', 999);
cart.addItem('Mouse', 29);

// Use credit card
cart.setPaymentStrategy(new CreditCardPayment('1234-5678-9012-3456', '123', '12/25'));
cart.checkout(); // Processing credit card payment of $1028

// Switch to PayPal
cart.setPaymentStrategy(new PayPalPayment('user@example.com', 'password'));
cart.checkout(); // Processing PayPal payment of $1028 from user@example.com
```

## Sorting Strategy Example

```typescript
// Strategy Interface
interface SortStrategy {
  sort(array: number[]): number[];
}

// Concrete Strategies
class BubbleSort implements SortStrategy {
  sort(array: number[]): number[] {
    const arr = [...array];
    const n = arr.length;
    for (let i = 0; i < n; i++) {
      for (let j = 0; j < n - i - 1; j++) {
        if (arr[j] > arr[j + 1]) {
          [arr[j], arr[j + 1]] = [arr[j + 1], arr[j]];
        }
      }
    }
    return arr;
  }
}

class QuickSort implements SortStrategy {
  sort(array: number[]): number[] {
    if (array.length <= 1) return array;
    
    const pivot = array[Math.floor(array.length / 2)];
    const left = array.filter(x => x < pivot);
    const middle = array.filter(x => x === pivot);
    const right = array.filter(x => x > pivot);
    
    return [
      ...this.sort(left),
      ...middle,
      ...this.sort(right)
    ];
  }
}

class MergeSort implements SortStrategy {
  sort(array: number[]): number[] {
    if (array.length <= 1) return array;
    
    const mid = Math.floor(array.length / 2);
    const left = this.sort(array.slice(0, mid));
    const right = this.sort(array.slice(mid));
    
    return this.merge(left, right);
  }

  private merge(left: number[], right: number[]): number[] {
    const result: number[] = [];
    let i = 0, j = 0;
    
    while (i < left.length && j < right.length) {
      if (left[i] <= right[j]) {
        result.push(left[i++]);
      } else {
        result.push(right[j++]);
      }
    }
    
    return [...result, ...left.slice(i), ...right.slice(j)];
  }
}

// Context
class DataSorter {
  constructor(private strategy: SortStrategy) {}

  setStrategy(strategy: SortStrategy): void {
    this.strategy = strategy;
  }

  sortData(data: number[]): number[] {
    return this.strategy.sort(data);
  }
}

// Usage
const data = [64, 34, 25, 12, 22, 11, 90];

const sorter = new DataSorter(new BubbleSort());
console.log(sorter.sortData(data)); // Bubble sort

sorter.setStrategy(new QuickSort());
console.log(sorter.sortData(data)); // Quick sort

sorter.setStrategy(new MergeSort());
console.log(sorter.sortData(data)); // Merge sort
```

## Real-World Example: Compression Strategies

```typescript
interface CompressionStrategy {
  compress(data: string): string;
  decompress(data: string): string;
}

class GzipCompression implements CompressionStrategy {
  compress(data: string): string {
    // Simulate Gzip compression
    return `GZIP[${data.split('').slice(0, 5).join('')}...]`;
  }

  decompress(data: string): string {
    // Simulate Gzip decompression
    return data.replace(/GZIP\[|\.\.\.\]/g, '');
  }
}

class DeflateCompression implements CompressionStrategy {
  compress(data: string): string {
    // Simulate Deflate compression
    return `DEFLATE[${data.split('').slice(0, 5).join('')}...]`;
  }

  decompress(data: string): string {
    return data.replace(/DEFLATE\[|\.\.\.\]/g, '');
  }
}

class BrotliCompression implements CompressionStrategy {
  compress(data: string): string {
    // Simulate Brotli compression
    return `BROTLI[${data.split('').slice(0, 5).join('')}...]`;
  }

  decompress(data: string): string {
    return data.replace(/BROTLI\[|\.\.\.\]/g, '');
  }
}

class FileCompressor {
  constructor(private strategy: CompressionStrategy) {}

  setCompressionStrategy(strategy: CompressionStrategy): void {
    this.strategy = strategy;
  }

  compressFile(filePath: string, content: string): { path: string; compressed: string } {
    const compressed = this.strategy.compress(content);
    return { path: `${filePath}.compressed`, compressed };
  }

  decompressFile(compressed: string): string {
    return this.strategy.decompress(compressed);
  }
}

// Usage
const fileContent = 'This is a large file with lots of content to compress...';
const compressor = new FileCompressor(new GzipCompression());

let result = compressor.compressFile('document.txt', fileContent);
console.log(result);

// Switch to Brotli
compressor.setCompressionStrategy(new BrotliCompression());
result = compressor.compressFile('document.txt', fileContent);
console.log(result);
```

## Dynamic Strategy Selection

```typescript
interface PaymentProcessor {
  process(amount: number): Promise<boolean>;
}

// Strategies
class CreditCardProcessor implements PaymentProcessor {
  async process(amount: number): Promise<boolean> {
    console.log(`Processing credit card payment: $${amount}`);
    await new Promise(resolve => setTimeout(resolve, 500));
    return Math.random() > 0.1; // 90% success rate
  }
}

class PayPalProcessor implements PaymentProcessor {
  async process(amount: number): Promise<boolean> {
    console.log(`Processing PayPal payment: $${amount}`);
    await new Promise(resolve => setTimeout(resolve, 1000));
    return Math.random() > 0.05; // 95% success rate
  }
}

// Context with strategy factory
class PaymentProcessor {
  private strategy: PaymentProcessor | null = null;

  selectStrategy(paymentMethod: 'creditcard' | 'paypal'): void {
    switch (paymentMethod) {
      case 'creditcard':
        this.strategy = new CreditCardProcessor();
        break;
      case 'paypal':
        this.strategy = new PayPalProcessor();
        break;
    }
  }

  async processPayment(amount: number, method: 'creditcard' | 'paypal'): Promise<boolean> {
    this.selectStrategy(method);
    if (!this.strategy) throw new Error('Invalid payment method');
    return this.strategy.process(amount);
  }
}

// Usage
const processor = new PaymentProcessor();
processor.processPayment(99.99, 'creditcard').then(success => {
  console.log(`Payment successful: ${success}`);
});
```

## Advantages

- **Easy to Add New Algorithms:** New strategies without modifying existing code
- **Eliminates Conditional Statements:** No more if/switch statements
- **Runtime Selection:** Change algorithm at runtime
- **Encapsulation:** Each algorithm is encapsulated
- **Single Responsibility:** Each strategy class has one reason to change
- **Open/Closed Principle:** Open for extension, closed for modification

## Disadvantages

- **Increased Number of Classes:** One class per strategy
- **Overhead for Simple Cases:** Overkill for simple algorithms
- **Complexity:** Can be complex with many strategies
- **Performance:** Extra method calls and indirection

## When to Use

- Multiple algorithms for one task
- Need to switch algorithms frequently
- Algorithms change or new ones are added often
- Avoid complex conditional logic
- Want algorithms to be independent of client code

## Best Practices

- **Use Common Interface:** All strategies implement same interface
- **Context Simplicity:** Context should be simple (just delegate)
- **Default Strategy:** Provide default strategy in context
- **Strategy Factory:** Use factory to create strategies
- **Documentation:** Document what each strategy does
- **Testing:** Test each strategy independently
- **Type Safety:** Use TypeScript for compile-time checking

## Related Patterns

- **Factory Pattern:** Create strategies
- **Decorator Pattern:** Combine with decorators for more behavior
- **Template Method:** Similar but used for inheritance instead of composition

## References & Sources

- Gang of Four - "Design Patterns" (Strategy Pattern)
- Refactoring.Guru - Strategy Pattern: https://refactoring.guru/design-patterns/strategy
- Christopher Alexander - Pattern Language
