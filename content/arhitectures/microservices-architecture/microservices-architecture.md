# Microservices Architecture

Microservices architecture is an approach to developing a single application as a suite of small, independently deployable services that communicate over the network. Each service is built around specific business capabilities and can be developed, deployed, and scaled independently.

## Key Concepts

- **Service:** Independent deployable unit focused on one business capability
- **Bounded Context:** Clear boundaries around service responsibilities
- **Service Discovery:** Mechanism to locate services dynamically
- **API Gateway:** Single entry point for all client requests
- **Inter-Service Communication:** Synchronous (REST, gRPC) or asynchronous (events, messages)
- **Data Isolation:** Each service manages its own data store
- **Independent Deployment:** Services can be deployed without affecting others
- **Resilience:** Fault isolation and recovery strategies

## Architecture Pattern

```
┌─────────────────────────────────────────────────────────────┐
│                      Client Applications                     │
└────────────────────────┬────────────────────────────────────┘
                         │
         ┌───────────────┴───────────────┐
         │     API Gateway / Mesh        │
         │  (Routing, Auth, Rate Limit)  │
         └───────────────┬───────────────┘
                         │
     ┌───────────────────┼───────────────────┐
     │                   │                   │
┌────▼────┐        ┌────▼────┐        ┌────▼────┐
│  Order   │        │ Inventory│       │ Payment  │
│ Service  │        │ Service  │       │ Service  │
└────┬────┘        └────┬────┘        └────┬────┘
     │                   │                   │
  ┌──▼──┐             ┌──▼──┐             ┌──▼──┐
  │ DB  │             │ DB  │             │ DB  │
  └─────┘             └─────┘             └─────┘
```

## Service Design

### Order Service

```typescript
// services/order/order.controller.ts
import { Router, Request, Response } from 'express';
import { OrderService } from './order.service';
import { ServiceRegistry } from '@/shared/registry';

export class OrderController {
  private orderService: OrderService;
  private inventoryService: InventoryServiceClient;
  private paymentService: PaymentServiceClient;

  constructor(registry: ServiceRegistry) {
    this.orderService = new OrderService();
    this.inventoryService = registry.getService('inventory');
    this.paymentService = registry.getService('payment');
  }

  async createOrder(req: Request, res: Response) {
    try {
      const { customerId, items } = req.body;

      // 1. Check inventory
      const available = await this.inventoryService.checkAvailability(items);
      if (!available) {
        return res.status(400).json({ error: 'Items not available' });
      }

      // 2. Create order
      const order = await this.orderService.createOrder({
        customerId,
        items,
        status: 'pending'
      });

      // 3. Process payment (async)
      this.paymentService.processPayment(order.id, order.total)
        .catch(error => {
          // Handle payment failure asynchronously
          this.orderService.updateOrderStatus(order.id, 'failed');
        });

      // 4. Reserve inventory
      await this.inventoryService.reserveItems(items, order.id);

      res.status(201).json(order);
    } catch (error) {
      res.status(500).json({ error: error.message });
    }
  }

  async getOrder(req: Request, res: Response) {
    const order = await this.orderService.getOrder(req.params.id);
    if (!order) {
      return res.status(404).json({ error: 'Order not found' });
    }
    res.json(order);
  }
}
```

### Service Structure

```
services/order/
├── order.controller.ts       # HTTP handlers
├── order.service.ts          # Business logic
├── order.repository.ts       # Data access
├── order.types.ts            # Types/DTOs
├── order.schema.ts           # Validation
├── db/
│   ├── connection.ts
│   └── migrations/
├── events/
│   ├── order-created.event.ts
│   └── order-confirmed.event.ts
├── clients/
│   ├── inventory.client.ts   # Inventory service client
│   ├── payment.client.ts     # Payment service client
│   └── notification.client.ts
└── app.ts
```

## Inter-Service Communication

### Synchronous (REST/gRPC)

```typescript
// services/order/clients/inventory.client.ts
import axios from 'axios';
import { CircuitBreaker } from '@/shared/resilience';

export class InventoryServiceClient {
  private client = axios.create({
    baseURL: process.env.INVENTORY_SERVICE_URL,
    timeout: 5000
  });

  private circuitBreaker = new CircuitBreaker({
    failureThreshold: 5,
    resetTimeout: 60000
  });

  async checkAvailability(items: OrderItem[]): Promise<boolean> {
    return this.circuitBreaker.execute(async () => {
      const response = await this.client.post('/check-availability', { items });
      return response.data.available;
    });
  }

  async reserveItems(items: OrderItem[], orderId: string): Promise<void> {
    return this.circuitBreaker.execute(async () => {
      await this.client.post('/reserve', { items, orderId });
    });
  }
}
```

### Asynchronous (Event-Driven)

```typescript
// services/order/events/order.publisher.ts
import { EventBus } from '@/shared/event-bus';

export class OrderEventPublisher {
  constructor(private eventBus: EventBus) {}

  async publishOrderCreated(order: Order): Promise<void> {
    await this.eventBus.publish('order.created', {
      orderId: order.id,
      customerId: order.customerId,
      items: order.items,
      total: order.total,
      timestamp: new Date()
    });
  }

  async publishOrderConfirmed(orderId: string): Promise<void> {
    await this.eventBus.publish('order.confirmed', {
      orderId,
      timestamp: new Date()
    });
  }

  async publishOrderCancelled(orderId: string): Promise<void> {
    await this.eventBus.publish('order.cancelled', {
      orderId,
      timestamp: new Date()
    });
  }
}

// services/order/events/order.subscriber.ts
export class OrderEventSubscriber {
  constructor(
    private eventBus: EventBus,
    private orderService: OrderService
  ) {}

  register(): void {
    // Listen to payment events
    this.eventBus.subscribe('payment.processed', async (event) => {
      await this.orderService.updateOrderStatus(
        event.orderId,
        'confirmed'
      );
    });

    this.eventBus.subscribe('payment.failed', async (event) => {
      await this.orderService.updateOrderStatus(
        event.orderId,
        'failed'
      );
    });

    // Listen to inventory events
    this.eventBus.subscribe('inventory.reserved', async (event) => {
      // Handle inventory confirmation
    });
  }
}
```

## API Gateway

Central entry point for all client requests:

```typescript
// api-gateway/gateway.ts
import express from 'express';
import { createProxyMiddleware } from 'express-http-proxy';
import { authMiddleware } from '@/shared/middleware/auth';
import { rateLimitMiddleware } from '@/shared/middleware/rate-limit';
import { loggingMiddleware } from '@/shared/middleware/logging';

export function createGateway() {
  const app = express();

  // Global middleware
  app.use(loggingMiddleware);
  app.use(authMiddleware);
  app.use(rateLimitMiddleware);

  // Route to services
  app.use('/api/orders', createProxyMiddleware({
    target: process.env.ORDER_SERVICE_URL,
    changeOrigin: true,
    pathRewrite: { '^/api/orders': '' },
    onError: (err, req, res) => {
      res.status(503).json({ error: 'Order service unavailable' });
    }
  }));

  app.use('/api/inventory', createProxyMiddleware({
    target: process.env.INVENTORY_SERVICE_URL,
    changeOrigin: true,
    pathRewrite: { '^/api/inventory': '' }
  }));

  app.use('/api/users', createProxyMiddleware({
    target: process.env.USER_SERVICE_URL,
    changeOrigin: true,
    pathRewrite: { '^/api/users': '' }
  }));

  return app;
}
```

## Service Discovery

### Static Configuration

```typescript
// config/services.config.ts
export const SERVICES = {
  order: {
    url: process.env.ORDER_SERVICE_URL || 'http://localhost:3001',
    timeout: 5000,
    retries: 3
  },
  inventory: {
    url: process.env.INVENTORY_SERVICE_URL || 'http://localhost:3002',
    timeout: 5000,
    retries: 3
  },
  payment: {
    url: process.env.PAYMENT_SERVICE_URL || 'http://localhost:3003',
    timeout: 10000,
    retries: 1
  },
  notification: {
    url: process.env.NOTIFICATION_SERVICE_URL || 'http://localhost:3004',
    timeout: 3000,
    retries: 3
  }
};
```

### Dynamic Discovery (Consul, Kubernetes)

```typescript
// services/registry.ts
import Consul from 'consul';

export class ServiceRegistry {
  private consul: Consul;

  constructor() {
    this.consul = new Consul({
      host: process.env.CONSUL_HOST || 'localhost',
      port: 8500
    });
  }

  async registerService(name: string, port: number): Promise<void> {
    await this.consul.agent.service.register({
      id: `${name}-${port}`,
      name,
      address: 'localhost',
      port,
      check: {
        http: `http://localhost:${port}/health`,
        interval: '10s',
        timeout: '5s'
      }
    });
  }

  async getService(name: string): Promise<{ address: string; port: number }> {
    const services = await this.consul.health.service({
      service: name,
      passing: true
    });

    if (services.length === 0) {
      throw new Error(`Service ${name} not available`);
    }

    const service = services[Math.floor(Math.random() * services.length)];
    return {
      address: service.Service.Address,
      port: service.Service.Port
    };
  }

  async deregisterService(serviceId: string): Promise<void> {
    await this.consul.agent.service.deregister(serviceId);
  }
}
```

## Data Management

### Database per Service

```
Order Service          Inventory Service       Payment Service
┌────────────┐         ┌────────────┐          ┌────────────┐
│   Order    │         │  Inventory │          │  Payment   │
│    DB      │         │     DB     │          │     DB     │
│ (PostgreSQL)         │ (PostgreSQL)           │ (PostgreSQL)
└────────────┘         └────────────┘          └────────────┘
```

### Data Consistency Patterns

#### Saga Pattern (Distributed Transactions)

```typescript
// Choreography-based saga
export class OrderSaga {
  constructor(
    private orderService: OrderService,
    private inventoryService: InventoryServiceClient,
    private paymentService: PaymentServiceClient,
    private eventBus: EventBus
  ) {
    this.registerEventHandlers();
  }

  private registerEventHandlers(): void {
    // Step 1: Order created
    this.eventBus.subscribe('order.created', async (order) => {
      try {
        // Step 2: Try to reserve inventory
        await this.inventoryService.reserveItems(order.items, order.id);
      } catch (error) {
        // Compensate: Cancel order
        await this.orderService.cancelOrder(order.id);
        await this.eventBus.publish('order.cancelled', { orderId: order.id });
      }
    });

    // Step 2: Inventory reserved
    this.eventBus.subscribe('inventory.reserved', async (event) => {
      try {
        // Step 3: Process payment
        await this.paymentService.processPayment(
          event.orderId,
          event.amount
        );
      } catch (error) {
        // Compensate: Release inventory
        await this.inventoryService.releaseItems(event.orderId);
        await this.orderService.cancelOrder(event.orderId);
      }
    });

    // Step 3: Payment processed
    this.eventBus.subscribe('payment.processed', async (event) => {
      // Step 4: Confirm order
      await this.orderService.confirmOrder(event.orderId);
      await this.eventBus.publish('order.confirmed', { orderId: event.orderId });
    });
  }
}
```

## Resilience Patterns

### Circuit Breaker

```typescript
export class CircuitBreaker {
  private failureCount = 0;
  private lastFailureTime = 0;
  private state: 'closed' | 'open' | 'half-open' = 'closed';

  constructor(
    private failureThreshold: number = 5,
    private resetTimeout: number = 60000
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'open') {
      if (Date.now() - this.lastFailureTime > this.resetTimeout) {
        this.state = 'half-open';
      } else {
        throw new Error('Circuit breaker is open');
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess(): void {
    this.failureCount = 0;
    this.state = 'closed';
  }

  private onFailure(): void {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    if (this.failureCount >= this.failureThreshold) {
      this.state = 'open';
    }
  }
}
```

### Retry with Exponential Backoff

```typescript
export async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  maxRetries: number = 3,
  baseDelay: number = 1000
): Promise<T> {
  let lastError: Error;

  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error as Error;
      if (i < maxRetries - 1) {
        const delay = baseDelay * Math.pow(2, i);
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
  }

  throw lastError!;
}
```

## Deployment

### Docker Compose (Development)

```yaml
version: '3.8'
services:
  api-gateway:
    build: ./api-gateway
    ports:
      - "3000:3000"
    environment:
      ORDER_SERVICE_URL: http://order-service:3001
      INVENTORY_SERVICE_URL: http://inventory-service:3002
      PAYMENT_SERVICE_URL: http://payment-service:3003
    depends_on:
      - order-service
      - inventory-service

  order-service:
    build: ./services/order
    ports:
      - "3001:3001"
    environment:
      DATABASE_URL: postgresql://user:pass@order-db:5432/order
      EVENT_BUS_URL: amqp://rabbitmq:5672
    depends_on:
      - order-db
      - rabbitmq

  inventory-service:
    build: ./services/inventory
    ports:
      - "3002:3002"
    environment:
      DATABASE_URL: postgresql://user:pass@inventory-db:5432/inventory

  order-db:
    image: postgres:15
    environment:
      POSTGRES_DB: order
      POSTGRES_PASSWORD: password
    volumes:
      - order-db-data:/var/lib/postgresql/data

  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"

volumes:
  order-db-data:
```

### Kubernetes Deployment

```yaml
# services/order/k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: myregistry.azurecr.io/order-service:v1
        ports:
        - containerPort: 3001
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: order-secrets
              key: database-url
        - name: EVENT_BUS_URL
          value: amqp://rabbitmq:5672
        livenessProbe:
          httpGet:
            path: /health
            port: 3001
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3001
```

## Monitoring & Observability

```typescript
// shared/monitoring.ts
import { trace, context } from '@opentelemetry/api';
import winston from 'winston';

const logger = winston.createLogger({
  format: winston.format.json(),
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
  ]
});

export function logRequest(req: Request, res: Response): void {
  logger.info({
    timestamp: new Date(),
    method: req.method,
    url: req.url,
    statusCode: res.statusCode,
    traceId: context.active().getValue('trace-id')
  });
}
```

## Advantages

- **Independent Scaling:** Scale services based on demand
- **Technology Diversity:** Use different tech stacks per service
- **Fault Isolation:** Service failure doesn't crash entire system
- **Faster Development:** Teams work independently on services
- **Easy Deployment:** Deploy services independently
- **Better Resource Utilization:** Scale only what's needed

## Disadvantages

- **Complexity:** Distributed system complexity
- **Network Latency:** Inter-service communication overhead
- **Data Consistency:** Eventual consistency challenges
- **Operational Overhead:** Monitoring, logging, deployment complexity
- **Testing Difficulty:** Integration testing is harder
- **Debugging:** Tracing issues across services is complex

## Best Practices

- **Clear Boundaries:** Define service boundaries around business capabilities
- **Independent Data:** Each service owns its data
- **Async Communication:** Use events for loose coupling
- **Circuit Breakers:** Implement resilience patterns
- **API Versioning:** Support multiple API versions
- **Logging & Tracing:** Centralized logging with correlation IDs
- **Health Checks:** Implement health check endpoints
- **Database Migrations:** Version and manage migrations per service
- **Documentation:** Document service contracts and APIs

## References & Sources

### Books
- **Sam Newman** - "Building Microservices: Designing Fine-Grained Systems" (2nd edition)
- **Chris Richardson** - "Microservices Patterns"
- **Newman, S., Nadareishvili, I., Mead, R., & Blanchard, M.** - "Building Microservices"

### Articles & Guides
- Martin Fowler - Microservices: https://martinfowler.com/articles/microservices.html
- Chris Richardson - Microservices Patterns: https://microservices.io/
- AWS - Microservices on AWS: https://aws.amazon.com/microservices/

### Technologies
- **Service Mesh:** Istio, Linkerd
- **Container Orchestration:** Kubernetes, Docker Swarm
- **API Gateway:** Kong, Nginx, AWS API Gateway
- **Message Brokers:** RabbitMQ, Apache Kafka, AWS SNS/SQS
- **Service Discovery:** Consul, Kubernetes, AWS Service Discovery
- **Monitoring:** Prometheus, Datadog, New Relic, ELK Stack

### Related Patterns
- Service Discovery
- API Gateway
- Circuit Breaker
- [Orchestration vs. Choreography Pattern](../../patterns/orchestration-choreography-pattern/orchestration-choreography-pattern.md) — the two strategies behind the Saga Pattern shown above
- [Event Sourcing Pattern](../../patterns/event-sourcing-pattern/event-sourcing-pattern.md)
- [CQRS Pattern](../../patterns/cqrs-pattern/cqrs-pattern.md)
- [Service-Oriented Architecture](../service-oriented-architecture/service-oriented-architecture.md) — the coarser-grained, centrally-integrated predecessor of this style
- [Distributed Systems Architecture](../distributed-architecture/distributed-architecture.md) — the broader style microservices is a concrete strategy within
- Database per Service
