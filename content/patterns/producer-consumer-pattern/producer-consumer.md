# Producer-Consumer Pattern

The Producer-Consumer Pattern is a concurrency design pattern where producers create data/tasks and place them in a shared buffer/queue, while consumers retrieve and process that data. This decouples producers from consumers, enabling independent scaling, rate control, and asynchronous processing.

## Problem

- Producers and consumers work at different speeds
- Direct coupling between producers and consumers
- Need to buffer work items
- Want to process items asynchronously
- Multiple producers/consumers need coordination

## Solution

Use a shared queue/buffer between producers and consumers to decouple them and enable independent operation.

## Structure

```
Producer 1 ──┐
Producer 2 ──┤─→ [Shared Queue/Buffer] ──→ Consumer 1
Producer 3 ──┘                        ──→ Consumer 2
                                       ──→ Consumer 3
```

## Basic Implementation with Queue

```typescript
// Task type
interface Task {
  id: string;
  data: any;
}

// Simple Queue
class TaskQueue {
  private queue: Task[] = [];
  private maxSize: number = 100;

  enqueue(task: Task): void {
    if (this.queue.length >= this.maxSize) {
      throw new Error('Queue is full');
    }
    this.queue.push(task);
    console.log(`Task ${task.id} added. Queue size: ${this.queue.length}`);
  }

  dequeue(): Task | null {
    if (this.queue.length === 0) {
      return null;
    }
    const task = this.queue.shift();
    console.log(`Task ${task?.id} removed. Queue size: ${this.queue.length}`);
    return task || null;
  }

  isEmpty(): boolean {
    return this.queue.length === 0;
  }

  isFull(): boolean {
    return this.queue.length >= this.maxSize;
  }

  size(): number {
    return this.queue.length;
  }
}

// Producer
class Producer {
  private taskCount = 0;

  constructor(
    private queue: TaskQueue,
    private name: string
  ) {}

  produce(count: number): void {
    for (let i = 0; i < count; i++) {
      this.taskCount++;
      const task: Task = {
        id: `${this.name}-${this.taskCount}`,
        data: Math.random()
      };

      while (this.queue.isFull()) {
        console.log(`${this.name} waiting... queue is full`);
        // In real scenario, would wait/block
      }

      this.queue.enqueue(task);
    }
  }
}

// Consumer
class Consumer {
  constructor(
    private queue: TaskQueue,
    private name: string
  ) {}

  async consume(processTime: number = 100): Promise<void> {
    while (true) {
      const task = this.queue.dequeue();

      if (task) {
        console.log(`${this.name} processing task: ${task.id}`);
        // Simulate processing
        await new Promise(resolve => setTimeout(resolve, processTime));
        console.log(`${this.name} finished task: ${task.id}`);
      } else {
        console.log(`${this.name} waiting... queue is empty`);
        // Wait and retry
        await new Promise(resolve => setTimeout(resolve, 100));
      }
    }
  }
}

// Usage
const queue = new TaskQueue();
const producer1 = new Producer(queue, 'Producer-1');
const producer2 = new Producer(queue, 'Producer-2');
const consumer1 = new Consumer(queue, 'Consumer-1');
const consumer2 = new Consumer(queue, 'Consumer-2');

producer1.produce(5);
producer2.produce(5);
consumer1.consume();
consumer2.consume();
```

## With Event Emitter Pattern

```typescript
import { EventEmitter } from 'events';

class EventBasedQueue extends EventEmitter {
  private queue: Task[] = [];
  private maxSize: number = 100;

  enqueue(task: Task): void {
    if (this.queue.length >= this.maxSize) {
      this.emit('queue:full');
      return;
    }
    this.queue.push(task);
    this.emit('task:added', task);
  }

  dequeue(): Task | null {
    if (this.queue.length === 0) {
      this.emit('queue:empty');
      return null;
    }
    const task = this.queue.shift();
    this.emit('task:removed', task);
    return task || null;
  }
}

// Producer
class EventProducer {
  constructor(
    private queue: EventBasedQueue,
    private name: string
  ) {
    this.queue.on('queue:full', () => {
      console.log(`${this.name} detected full queue`);
    });
  }

  produce(task: Task): void {
    this.queue.enqueue(task);
  }
}

// Consumer
class EventConsumer {
  constructor(
    private queue: EventBasedQueue,
    private name: string
  ) {
    this.queue.on('task:added', (task: Task) => {
      this.process(task);
    });
  }

  private async process(task: Task): Promise<void> {
    console.log(`${this.name} processing: ${task.id}`);
    await new Promise(resolve => setTimeout(resolve, 100));
    console.log(`${this.name} completed: ${task.id}`);
  }
}

// Usage
const queue = new EventBasedQueue();
const producer = new EventProducer(queue, 'Producer');
const consumer = new EventConsumer(queue, 'Consumer');

producer.produce({ id: 'task-1', data: {} });
producer.produce({ id: 'task-2', data: {} });
```

## With RxJS (Reactive)

```typescript
import { Subject, merge } from 'rxjs';
import { buffer, bufferTime, tap } from 'rxjs/operators';

class RxProducerConsumer {
  private taskSubject = new Subject<Task>();
  private tasks$ = this.taskSubject.asObservable();

  // Producer: Generate tasks
  produceTask(task: Task): void {
    this.taskSubject.next(task);
  }

  // Consumer: Process tasks in batches
  startConsumer(batchSize: number = 5): void {
    this.tasks$
      .pipe(
        buffer(
          this.tasks$.pipe(
            bufferTime(1000) // Or use another observable
          )
        ),
        tap(tasks => {
          if (tasks.length > 0) {
            console.log(`Processing batch of ${tasks.length} tasks`);
            tasks.forEach(task => {
              console.log(`  - Processing: ${task.id}`);
            });
          }
        })
      )
      .subscribe();
  }
}

// Usage
const rxQueue = new RxProducerConsumer();
rxQueue.startConsumer(5);

for (let i = 0; i < 10; i++) {
  rxQueue.produceTask({ id: `task-${i}`, data: {} });
}
```

## Real-World Example: Task Processing System

```typescript
interface WorkItem {
  id: string;
  type: 'email' | 'notification' | 'report';
  data: any;
  priority: number;
  retries: number;
}

class PriorityQueue {
  private items: WorkItem[] = [];

  enqueue(item: WorkItem): void {
    this.items.push(item);
    this.items.sort((a, b) => b.priority - a.priority);
  }

  dequeue(): WorkItem | null {
    return this.items.shift() || null;
  }

  isEmpty(): boolean {
    return this.items.length === 0;
  }
}

class TaskProducer {
  constructor(private queue: PriorityQueue) {}

  submitEmailTask(email: string, subject: string, priority: number = 1): void {
    const task: WorkItem = {
      id: `email-${Date.now()}`,
      type: 'email',
      data: { email, subject },
      priority,
      retries: 0
    };
    this.queue.enqueue(task);
  }

  submitNotificationTask(userId: string, message: string, priority: number = 2): void {
    const task: WorkItem = {
      id: `notif-${Date.now()}`,
      type: 'notification',
      data: { userId, message },
      priority,
      retries: 0
    };
    this.queue.enqueue(task);
  }
}

class TaskConsumer {
  constructor(
    private queue: PriorityQueue,
    private name: string,
    private maxRetries: number = 3
  ) {}

  async start(): Promise<void> {
    while (true) {
      const task = this.queue.dequeue();

      if (!task) {
        await new Promise(resolve => setTimeout(resolve, 1000));
        continue;
      }

      try {
        console.log(`${this.name} processing: ${task.id} (priority: ${task.priority})`);
        await this.executeTask(task);
        console.log(`${this.name} completed: ${task.id}`);
      } catch (error) {
        if (task.retries < this.maxRetries) {
          task.retries++;
          console.log(`${this.name} retrying: ${task.id} (attempt ${task.retries})`);
          this.queue.enqueue(task);
        } else {
          console.error(`${this.name} failed: ${task.id} (max retries exceeded)`);
        }
      }
    }
  }

  private async executeTask(task: WorkItem): Promise<void> {
    await new Promise(resolve => setTimeout(resolve, 100)); // Simulate work
    if (Math.random() < 0.1) throw new Error('Random failure');
  }
}

// Usage
const queue = new PriorityQueue();
const producer = new TaskProducer(queue);
const consumer1 = new TaskConsumer(queue, 'Consumer-1');
const consumer2 = new TaskConsumer(queue, 'Consumer-2');

// Produce tasks
producer.submitEmailTask('user@example.com', 'Welcome', 1);
producer.submitNotificationTask('user123', 'New message', 2);
producer.submitEmailTask('admin@example.com', 'Report', 3);

// Start consumers
consumer1.start();
consumer2.start();
```

## Advantages

- **Decoupling:** Producers and consumers are independent
- **Rate Control:** Buffer handles speed differences
- **Asynchronous Processing:** Non-blocking operations
- **Scalability:** Add/remove producers and consumers easily
- **Load Balancing:** Multiple consumers handle load
- **Flexibility:** Different processing speeds supported

## Disadvantages

- **Complexity:** More code and coordination logic
- **Memory Overhead:** Queue storage for buffered tasks
- **Latency:** Additional delay due to queuing
- **Error Handling:** Failed tasks need retry logic
- **Ordering Issues:** May not preserve order with multiple consumers

## Best Practices

- **Bounded Queue:** Prevent unbounded memory usage
- **Priority Queue:** Process critical items first
- **Retry Logic:** Handle failed tasks gracefully
- **Monitoring:** Track queue size and processing rate
- **Graceful Shutdown:** Complete pending tasks on shutdown
- **Thread Safety:** Use thread-safe queues in multi-threaded environments
- **Backpressure:** Handle producer faster than consumer scenario

## Related Patterns

- **[Observer Pattern](../observer-pattern/observer-pattern.md):** Similar event-driven approach
- **Mediator Pattern:** Queue acts as mediator
- **Thread Pool Pattern:** Consumers can be thread pools
- **[Pub/Sub Pattern](../pub-sub-pattern/pub-sub-pattern.md):** Similar with multiple topics
- **[Concurrency Architecture](../../arhitectures/concurrency-architecture/concurrency-architecture.md):** The broader architectural style this pattern implements

## References & Sources

- Producer-Consumer Pattern: https://en.wikipedia.org/wiki/Producer%E2%80%93consumer_problem
- RxJS Buffers: https://rxjs.dev/guide/operators
- Node.js EventEmitter: https://nodejs.org/api/events.html
- Java BlockingQueue: https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/BlockingQueue.html
