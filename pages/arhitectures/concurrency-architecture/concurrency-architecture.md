# Concurrency Architecture

Concurrency Architecture organizes a system around executing multiple tasks simultaneously, or interleaved over overlapping time, so independent units of work make progress without corrupting shared state. It's concerned with how work is divided, scheduled, and synchronized — through threads, processes, actors, or an event loop — rather than with what the work actually does.

## Key Concepts

- **Concurrency vs. Parallelism:** Concurrency is structuring a program to handle multiple tasks in overlapping time (even on one core); parallelism is literally running tasks at the same instant on multiple cores
- **Thread/Process:** Independent unit of execution; threads share memory with the parent process, processes don't
- **Event Loop:** Single-threaded concurrency achieved by scheduling non-blocking callbacks instead of using multiple threads (Node.js's model)
- **Actor Model:** Independent actors that only interact by sending messages to each other's mailbox — no shared mutable state
- **Race Condition:** Bug caused by outcome depending on unpredictable timing of concurrent operations
- **Lock/Mutex:** Mechanism that grants exclusive access to shared state to prevent race conditions
- **Worker Pool:** A fixed set of workers pulling tasks from a shared queue, bounding concurrency to available resources
- **Backpressure:** Slowing down producers when consumers can't keep up, to avoid unbounded queue growth

## Architecture Diagram

```
Worker Pool Model                       Actor Model
┌──────────────┐                        ┌───────┐  message  ┌───────┐
│  Task Queue   │                       │Actor A│──────────▶│Actor B│
└──┬──┬──┬──┬──┘                        └───┬───┘           └───┬───┘
   ▼  ▼  ▼  ▼                                │ mailbox            │ mailbox
 W1 W2 W3 W4   ← fixed pool of workers        ▼                    ▼
                                        (processes one message at a time,
                                         no shared mutable state)
```

## Example: Worker Pool for CPU-Bound Work

```typescript
// worker-pool.ts (main thread)
import { Worker } from 'node:worker_threads';
import os from 'node:os';

interface Task {
  data: number[];
  resolve: (result: number) => void;
  reject: (error: Error) => void;
}

class WorkerPool {
  private workers: Worker[] = [];
  private idleWorkers: Worker[] = [];
  private queue: Task[] = [];

  constructor(workerScript: string, size = os.cpus().length) {
    for (let i = 0; i < size; i++) {
      const worker = new Worker(workerScript);
      this.workers.push(worker);
      this.idleWorkers.push(worker);
    }
  }

  run(data: number[]): Promise<number> {
    return new Promise((resolve, reject) => {
      this.queue.push({ data, resolve, reject });
      this.dispatch();
    });
  }

  private dispatch(): void {
    if (this.queue.length === 0 || this.idleWorkers.length === 0) return;

    const worker = this.idleWorkers.pop()!;
    const task = this.queue.shift()!;

    const onMessage = (result: number) => {
      cleanup();
      task.resolve(result);
      this.idleWorkers.push(worker);
      this.dispatch(); // pick up the next queued task
    };
    const onError = (error: Error) => {
      cleanup();
      task.reject(error);
      this.idleWorkers.push(worker);
      this.dispatch();
    };
    const cleanup = () => {
      worker.off('message', onMessage);
      worker.off('error', onError);
    };

    worker.once('message', onMessage);
    worker.once('error', onError);
    worker.postMessage(task.data);
  }
}

// Usage: distribute CPU-heavy work across all cores instead of blocking the event loop
const pool = new WorkerPool('./sum-worker.js');
const results = await Promise.all([
  pool.run([1, 2, 3, 4, 5]),
  pool.run([10, 20, 30]),
  pool.run([100, 200])
]);
```

See [Node.js Thread Workers](../../runtimes/nodejs/thread-workers.md) for the full `worker_threads` API.

## Example: Actor-Style Message Passing

```typescript
// No shared mutable state — actors only communicate through messages in their mailbox,
// which eliminates entire classes of race conditions by construction.
type Message = { type: string; payload: unknown };

abstract class Actor {
  private mailbox: Message[] = [];
  private processing = false;

  send(message: Message): void {
    this.mailbox.push(message);
    this.processNext();
  }

  private async processNext(): Promise<void> {
    if (this.processing || this.mailbox.length === 0) return;
    this.processing = true;
    const message = this.mailbox.shift()!;
    await this.receive(message);
    this.processing = false;
    this.processNext();
  }

  protected abstract receive(message: Message): Promise<void>;
}

class InventoryActor extends Actor {
  private stock = new Map<string, number>([['sku-1', 10]]);

  protected async receive(message: Message): Promise<void> {
    if (message.type === 'RESERVE') {
      const { sku, quantity, reply } = message.payload as {
        sku: string; quantity: number; reply: (ok: boolean) => void;
      };
      const available = this.stock.get(sku) ?? 0;
      if (available >= quantity) {
        this.stock.set(sku, available - quantity);
        reply(true);
      } else {
        reply(false);
      }
    }
  }
}

// Usage
const inventory = new InventoryActor();
inventory.send({
  type: 'RESERVE',
  payload: { sku: 'sku-1', quantity: 3, reply: (ok: boolean) => console.log('Reserved:', ok) }
});
```

## Advantages

- **Throughput:** Independent tasks make progress in parallel instead of waiting on each other
- **Responsiveness:** An event loop or worker pool keeps the main thread free to handle new requests
- **Resource Utilization:** Multi-core hardware is actually used instead of sitting idle
- **Fault Isolation (Actor Model):** A crashed actor doesn't corrupt another actor's state, since none is shared

## Disadvantages

- **Race Conditions:** Shared mutable state accessed from multiple threads without synchronization produces nondeterministic bugs
- **Deadlocks:** Two or more tasks waiting on each other's locks can freeze the system entirely
- **Debugging Difficulty:** Concurrency bugs are often timing-dependent and hard to reproduce
- **Coordination Overhead:** Locks, message passing, and queues all add code and latency compared to sequential logic

## When to Use

- CPU-bound work that would otherwise block a single-threaded event loop
- I/O-bound workloads with many simultaneous slow operations (network calls, file access)
- Systems needing fault isolation between independent units of work (actor model)
- Any workload that can be split into independent, parallelizable chunks

## Best Practices

- **Prefer Message Passing Over Shared State:** Eliminates whole categories of race conditions
- **Bound Your Concurrency:** Use a fixed-size worker pool or semaphore instead of unbounded parallelism
- **Make Operations Idempotent:** Simplifies retries when tasks run concurrently
- **Isolate Mutable State:** If state must be shared, guard every access with a lock/mutex consistently
- **Design for Backpressure:** Slow consumers must be able to signal producers to slow down

## Common Mistakes

- **Unsynchronized Shared State:** Multiple concurrent writers touching the same object with no lock or atomic operation
- **Unbounded Concurrency:** Spawning a task per incoming item with no cap, exhausting memory or connections under load
- **Blocking the Event Loop:** Running CPU-heavy synchronous code on Node.js's single thread instead of offloading to a worker
- **Ignoring Deadlock Potential:** Acquiring multiple locks in inconsistent order across different code paths

## Related Patterns

- [Producer-Consumer Pattern](../../patterns/producer-consumer-pattern/producer-consumer.md) — the queueing mechanism underlying most worker-pool designs
- [Event-Driven Architecture](../event-driven-architecture/event-driven-architecture.md) — concurrency expressed through asynchronous events instead of threads
- [Distributed Systems Architecture](../distributed-architecture/distributed-architecture.md) — concurrency problems recur across machines, not just within one process
- [Node.js Thread Workers](../../runtimes/nodejs/thread-workers.md) and [Node.js Event Loop](../../runtimes/nodejs/event-loop.md)

## References & Sources

- Maurice Herlihy & Nir Shavit — "The Art of Multiprocessor Programming" (2008)
- Carl Hewitt, Peter Bishop, Richard Steiger — "A Universal Modular Actor Formalism for Artificial Intelligence" (1973), origin of the Actor Model
- Node.js — Worker Threads: https://nodejs.org/api/worker_threads.html
- Go — Concurrency Patterns (CSP model): https://go.dev/blog/pipelines
