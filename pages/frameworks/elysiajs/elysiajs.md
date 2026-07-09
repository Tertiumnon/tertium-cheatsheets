# ElysiaJS

ElysiaJS is a modern, fast, type-safe backend framework built for Bun runtime. It emphasizes end-to-end type safety, performance, and developer experience with TypeScript-first design.

## Key Features

- **Type Safety:** End-to-end type inference from routes to responses
- **Performance:** Built on Bun for exceptional speed
- **TypeScript-First:** Full TypeScript support with type inference
- **Validation:** Built-in schema validation with type extraction
- **Decorator Support:** Clean syntax with TypeScript decorators
- **Microservices Ready:** RPC and microservice patterns built-in
- **Plugin System:** Extensible architecture for custom plugins
- **Zero Config:** Works out of the box with sensible defaults

## Installation

```bash
# Using Bun
bun add elysia

# Start new project
bun create elysia app
cd app
bun run dev
```

## Basic Server Setup

### Simple Hello World

```typescript
import Elysia from 'elysia';

const app = new Elysia()
  .get('/', () => 'Hello, World!')
  .listen(3000);

console.log(`Server running at http://${app.server?.hostname}:${app.server?.port}`);
```

### With Type Inference

```typescript
import Elysia from 'elysia';

const app = new Elysia()
  .get('/users/:id', ({ params: { id } }) => {
    // 'id' is automatically inferred as string
    return { userId: id };
  })
  .post('/users', ({ body }) => {
    // 'body' is inferred from route handler
    return { id: 1, ...body };
  })
  .listen(3000);
```

## Routing & Methods

### HTTP Methods

```typescript
import Elysia from 'elysia';

const app = new Elysia()
  // GET
  .get('/posts', () => ({ posts: [] }))
  
  // POST
  .post('/posts', ({ body }) => ({ id: 1, ...body }))
  
  // PUT
  .put('/posts/:id', ({ params, body }) => ({ id: params.id, ...body }))
  
  // PATCH
  .patch('/posts/:id', ({ params, body }) => ({ id: params.id, ...body }))
  
  // DELETE
  .delete('/posts/:id', ({ params }) => ({ deleted: params.id }))
  
  // HEAD
  .head('/posts', () => null)
  
  // OPTIONS
  .options('/posts', () => null)
  
  .listen(3000);
```

### Route Grouping

```typescript
import Elysia from 'elysia';

const api = new Elysia({ prefix: '/api' })
  .get('/users', () => ({ users: [] }))
  .post('/users', ({ body }) => body);

const app = new Elysia()
  .use(api)
  .get('/', () => 'Home')
  .listen(3000);

// Routes: GET /api/users, POST /api/users, GET /
```

### Dynamic Routes

```typescript
const app = new Elysia()
  // Single parameter
  .get('/users/:id', ({ params: { id } }) => ({ userId: id }))
  
  // Multiple parameters
  .get('/posts/:postId/comments/:commentId', 
    ({ params: { postId, commentId } }) => ({ postId, commentId })
  )
  
  // Optional parameter (with regex)
  .get('/files/:name?', ({ params: { name } }) => ({ filename: name }))
  
  .listen(3000);
```

## Schema & Validation

### Using Elysia.t (Built-in Validator)

```typescript
import Elysia, { t } from 'elysia';

const CreateUserSchema = t.Object({
  name: t.String({ minLength: 1 }),
  email: t.String({ format: 'email' }),
  age: t.Optional(t.Number({ minimum: 0 }))
});

type CreateUserDTO = typeof CreateUserSchema.static;

const app = new Elysia()
  .post('/users', 
    ({ body }) => {
      // body is automatically typed as CreateUserDTO
      return { id: 1, ...body };
    },
    {
      body: CreateUserSchema
    }
  )
  .listen(3000);
```

### Schema with Constraints

```typescript
import Elysia, { t } from 'elysia';

const UserSchema = t.Object({
  id: t.Number(),
  email: t.String({ 
    format: 'email',
    description: 'User email address'
  }),
  age: t.Number({ 
    minimum: 0, 
    maximum: 150,
    description: 'User age in years'
  }),
  role: t.Union([
    t.Literal('admin'),
    t.Literal('user'),
    t.Literal('guest')
  ]),
  tags: t.Array(t.String()),
  metadata: t.Optional(t.Record(t.String(), t.Unknown()))
});

const app = new Elysia()
  .post('/users', ({ body }) => body, {
    body: UserSchema,
    detail: { 
      summary: 'Create new user',
      tags: ['Users']
    }
  })
  .listen(3000);
```

## Context & State

### Request Context

```typescript
import Elysia from 'elysia';

const app = new Elysia()
  .get('/request-info', ({ request, headers, query }) => {
    return {
      method: request.method,
      url: request.url,
      headers: Object.fromEntries(headers),
      query: query
    };
  })
  .listen(3000);
```

### Global State

```typescript
import Elysia from 'elysia';

interface AppState {
  counter: number;
  users: string[];
}

const app = new Elysia()
  .state<AppState>({
    counter: 0,
    users: []
  })
  .get('/counter', ({ store }) => ({
    count: store.counter
  }))
  .post('/increment', ({ store }) => ({
    count: ++store.counter
  }))
  .post('/users', ({ body, store }) => {
    store.users.push(body.name);
    return { users: store.users };
  }, {
    body: { name: 'string' }
  })
  .listen(3000);
```

## Middleware & Guards

### Guard (Validation Middleware)

```typescript
import Elysia from 'elysia';

const app = new Elysia()
  .guard({
    headers: { 
      'x-api-key': 'string'
    }
  }, (app) =>
    app.get('/protected', ({ headers }) => ({
      message: 'Protected route',
      apiKey: headers['x-api-key']
    }))
  )
  .listen(3000);
```

### Custom Middleware

```typescript
import Elysia from 'elysia';

const app = new Elysia()
  .derive(({ headers }) => {
    const startTime = Date.now();
    
    return {
      startTime,
      userId: headers['x-user-id']
    };
  })
  .get('/user-request', ({ userId, startTime }) => {
    const duration = Date.now() - startTime;
    return { userId, duration };
  })
  .listen(3000);
```

### Logging Middleware

```typescript
import Elysia from 'elysia';

const loggingPlugin = (app: Elysia) => {
  return app
    .derive(({ request }) => {
      console.log(`[${new Date().toISOString()}] ${request.method} ${request.url}`);
      return {};
    });
};

const app = new Elysia()
  .use(loggingPlugin)
  .get('/posts', () => ({ posts: [] }))
  .listen(3000);
```

## Error Handling

### Error Responses

```typescript
import Elysia, { error } from 'elysia';

const app = new Elysia()
  .get('/users/:id', ({ params: { id } }) => {
    if (id === '0') {
      return error(400, 'Invalid user ID');
    }
    if (id === 'notfound') {
      return error(404, 'User not found');
    }
    return { id };
  })
  .listen(3000);
```

### Error Handler

```typescript
import Elysia from 'elysia';

const app = new Elysia()
  .onError(({ error, code }) => {
    console.error(`Error [${code}]:`, error.message);
    
    if (code === 'NOT_FOUND') {
      return { error: 'Route not found' };
    }
    
    if (code === 'VALIDATION') {
      return { error: 'Validation failed' };
    }
    
    return { error: 'Internal server error' };
  })
  .get('/posts/:id', ({ params: { id } }) => {
    if (!id) throw new Error('ID is required');
    return { id };
  })
  .listen(3000);
```

## Plugins & Modules

### Creating a Plugin

```typescript
// plugins/users.plugin.ts
import Elysia, { t } from 'elysia';

export const userPlugin = (app: Elysia) => {
  return app
    .state('users', new Map())
    .get('/users', ({ store }) => {
      return Array.from(store.users.values());
    })
    .post('/users', ({ body, store }) => {
      const id = store.users.size + 1;
      store.users.set(id, body);
      return { id, ...body };
    }, {
      body: t.Object({
        name: t.String(),
        email: t.String({ format: 'email' })
      })
    })
    .get('/users/:id', ({ params: { id }, store }) => {
      const user = store.users.get(Number(id));
      if (!user) return error(404, 'User not found');
      return { id, ...user };
    });
};

// app.ts
import Elysia from 'elysia';
import { userPlugin } from './plugins/users.plugin';

const app = new Elysia()
  .use(userPlugin)
  .listen(3000);
```

### Feature Modules

```typescript
// modules/auth/auth.module.ts
import Elysia, { t } from 'elysia';
import { jwt } from '@elysiajs/jwt';

export const authModule = new Elysia({ prefix: '/auth' })
  .use(jwt({
    name: 'jwt',
    secret: process.env.JWT_SECRET!
  }))
  .post('/login', async ({ body, jwt: jwtService }) => {
    // Verify credentials
    const token = await jwtService.sign({ userId: 1 });
    return { token };
  }, {
    body: t.Object({
      email: t.String({ format: 'email' }),
      password: t.String()
    })
  })
  .post('/verify', async ({ headers, jwt: jwtService }) => {
    const token = headers['authorization']?.split(' ')[1];
    if (!token) return error(401, 'No token provided');
    
    const verified = await jwtService.verify(token);
    return { verified };
  });

// app.ts
import Elysia from 'elysia';
import { authModule } from './modules/auth/auth.module';

const app = new Elysia()
  .use(authModule)
  .listen(3000);
```

## Project Structure

```
src/
├── modules/                          # Feature modules
│   ├── auth/
│   │   ├── auth.controller.ts        # Routes
│   │   ├── auth.service.ts           # Business logic
│   │   ├── auth.guard.ts             # Auth middleware
│   │   └── auth.module.ts            # Module export
│   ├── users/
│   │   ├── users.controller.ts
│   │   ├── users.service.ts
│   │   ├── users.types.ts
│   │   └── users.module.ts
│   └── posts/
│       └── ...
├── middleware/                       # Global middleware
│   ├── logging.middleware.ts
│   ├── error-handler.middleware.ts
│   └── cors.middleware.ts
├── services/                         # Shared services
│   ├── database.service.ts
│   ├── cache.service.ts
│   └── logger.service.ts
├── types/                            # Shared types
│   ├── index.ts
│   └── api.types.ts
├── utils/                            # Utilities
│   ├── validators.ts
│   └── helpers.ts
├── app.ts                            # App configuration
└── server.ts                         # Entry point
```

## Common Plugins

### JWT Authentication

```typescript
import { jwt } from '@elysiajs/jwt';
import Elysia from 'elysia';

const app = new Elysia()
  .use(jwt({
    name: 'jwt',
    secret: process.env.JWT_SECRET!
  }))
  .post('/login', async ({ jwt: jwtService }) => {
    const token = await jwtService.sign({ userId: 1 });
    return { token };
  })
  .get('/profile', async ({ jwt: jwtService, headers }) => {
    const token = headers['authorization']?.split(' ')[1];
    const verified = await jwtService.verify(token);
    return { verified };
  }, {
    headers: { authorization: 'string' }
  })
  .listen(3000);
```

### CORS

```typescript
import { cors } from '@elysiajs/cors';
import Elysia from 'elysia';

const app = new Elysia()
  .use(cors({
    origin: ['http://localhost:3000', 'http://localhost:5173'],
    credentials: true,
    methods: ['GET', 'POST', 'PUT', 'DELETE']
  }))
  .get('/data', () => ({ data: [] }))
  .listen(3000);
```

### Swagger Documentation

```typescript
import { swagger } from '@elysiajs/swagger';
import Elysia, { t } from 'elysia';

const app = new Elysia()
  .use(swagger({
    documentation: {
      info: {
        title: 'My API',
        version: '1.0.0'
      }
    }
  }))
  .get('/posts', () => ({ posts: [] }), {
    detail: { 
      summary: 'Get all posts',
      tags: ['Posts']
    }
  })
  .post('/posts', ({ body }) => body, {
    body: t.Object({
      title: t.String(),
      content: t.String()
    }),
    detail: {
      summary: 'Create new post',
      tags: ['Posts']
    }
  })
  .listen(3000);
```

### Database Integration (Prisma)

```typescript
import Elysia, { t } from 'elysia';
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

const app = new Elysia()
  .state('db', prisma)
  .get('/users', async ({ store }) => {
    return store.db.user.findMany();
  })
  .post('/users', async ({ body, store }) => {
    return store.db.user.create({ data: body });
  }, {
    body: t.Object({
      name: t.String(),
      email: t.String({ format: 'email' })
    })
  })
  .listen(3000);
```

## Performance Tips

- **Use schema validation:** Catches errors early
- **Leverage type inference:** Reduces runtime errors
- **Stream responses:** For large data
- **Cache frequently accessed data:** Use state or external cache
- **Use guards for validation:** Pre-filter requests
- **Deploy with Bun:** Maximum performance gains

## Best Practices

- **Organize by feature:** Group related routes and logic
- **Use plugins for modularity:** Keep code DRY
- **Type everything:** Leverage TypeScript for safety
- **Validate at boundaries:** Use schemas for all inputs
- **Error handling:** Implement comprehensive error handlers
- **Logging:** Track requests and errors
- **Environment variables:** Use .env files
- **Testing:** Test routes and middleware separately

## References & Sources

### Official Documentation
- ElysiaJS Official Docs: https://elysiajs.com/
- ElysiaJS GitHub: https://github.com/elysiajs/elysia
- Bun Official Docs: https://bun.sh/

### Plugins & Extensions
- ElysiaJS Plugins: https://elysiajs.com/plugins
- JWT Plugin: https://elysiajs.com/plugins/jwt
- CORS Plugin: https://elysiajs.com/plugins/cors
- Swagger Plugin: https://elysiajs.com/plugins/swagger

### Related Technologies
- **Bun:** TypeScript-first runtime for JavaScript
- **TypeScript:** Type-safe JavaScript
- **Prisma:** ORM for Node.js and Bun
- **Zod:** TypeScript-first schema validation (alternative)

### Concepts
- RESTful API Design
- Type-Safe Backend Development
- Microservices Architecture
- Plugin-Based Architecture

### Note
This guide covers ElysiaJS best practices and patterns. ElysiaJS emphasizes end-to-end type safety and performance, making it ideal for building type-safe APIs with Bun.
