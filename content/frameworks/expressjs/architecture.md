# Express.js Architecture

Express.js is a minimal and flexible Node.js web application framework. This guide covers organizing large-scale Express applications with clean architecture, separation of concerns, and scalability.

## Key Principles

- **Middleware Stack:** Compose functionality with middleware
- **Modular Routing:** Organize routes by feature
- **Layer Separation:** Distinct request, service, and data layers
- **Error Handling:** Centralized error handling middleware
- **Configuration Management:** Environment-based configuration
- **Dependency Injection:** Pass dependencies to controllers and services

## Project Structure (Feature-Based)

Organize by business domains/features with clear layers:

```
src/
├── routes/                           # API route definitions
│   ├── index.ts                      # Route aggregator
│   ├── users.routes.ts
│   ├── posts.routes.ts
│   └── auth.routes.ts
├── controllers/                      # Request handlers
│   ├── users.controller.ts
│   ├── posts.controller.ts
│   └── auth.controller.ts
├── services/                         # Business logic
│   ├── users.service.ts
│   ├── posts.service.ts
│   ├── auth.service.ts
│   ├── email.service.ts
│   ├── cache.service.ts
│   └── logger.service.ts
├── repositories/                     # Data access layer
│   ├── users.repository.ts
│   ├── posts.repository.ts
│   ├── base.repository.ts            # Base class for all repositories
│   └── index.ts
├── middleware/                       # Express middleware
│   ├── auth.middleware.ts
│   ├── validation.middleware.ts
│   ├── error-handler.middleware.ts
│   ├── logging.middleware.ts
│   ├── cors.middleware.ts
│   └── index.ts
├── validators/                       # Request validation
│   ├── users.validator.ts
│   ├── posts.validator.ts
│   └── common.validator.ts
├── types/                            # TypeScript types/interfaces
│   ├── index.ts
│   ├── api.types.ts
│   ├── dto.types.ts
│   ├── errors.types.ts
│   └── express.types.ts
├── utils/                            # Utility functions
│   ├── helpers.ts
│   ├── formatters.ts
│   ├── validators.ts
│   └── constants.ts
├── config/                           # Configuration files
│   ├── database.config.ts
│   ├── cache.config.ts
│   ├── app.config.ts
│   └── env.ts                        # Environment variables
├── database/                         # Database setup
│   ├── connection.ts
│   ├── migrations/
│   └── seeds/
├── app.ts                            # Express app setup
└── server.ts                         # Entry point
```

## Alternative: Modular Feature-Based Structure

```
src/
├── modules/                          # Feature modules (recommended for large apps)
│   ├── auth/
│   │   ├── auth.controller.ts
│   │   ├── auth.service.ts
│   │   ├── auth.routes.ts
│   │   ├── auth.middleware.ts
│   │   ├── auth.types.ts
│   │   └── auth.validator.ts
│   ├── users/
│   │   ├── users.controller.ts
│   │   ├── users.service.ts
│   │   ├── users.repository.ts
│   │   ├── users.routes.ts
│   │   ├── users.types.ts
│   │   └── users.validator.ts
│   ├── posts/
│   │   └── ...
│   └── index.ts
├── common/                           # Shared across modules
│   ├── middleware/
│   ├── utils/
│   ├── types/
│   └── validators/
├── config/
├── database/
├── app.ts
└── server.ts
```

## Controller Layer

```typescript
// src/controllers/users.controller.ts
import { Request, Response, NextFunction } from 'express';
import { UsersService } from '../services/users.service';

export class UsersController {
  constructor(private usersService: UsersService) {}

  async getAllUsers(req: Request, res: Response, next: NextFunction) {
    try {
      const { limit = 10, offset = 0 } = req.query;
      const users = await this.usersService.getAllUsers(
        Number(limit),
        Number(offset)
      );
      res.json({
        success: true,
        data: users
      });
    } catch (error) {
      next(error);
    }
  }

  async getUserById(req: Request, res: Response, next: NextFunction) {
    try {
      const { id } = req.params;
      const user = await this.usersService.getUserById(Number(id));
      
      if (!user) {
        return res.status(404).json({
          success: false,
          error: 'User not found'
        });
      }

      res.json({
        success: true,
        data: user
      });
    } catch (error) {
      next(error);
    }
  }

  async createUser(req: Request, res: Response, next: NextFunction) {
    try {
      const user = await this.usersService.createUser(req.body);
      res.status(201).json({
        success: true,
        data: user
      });
    } catch (error) {
      next(error);
    }
  }

  async updateUser(req: Request, res: Response, next: NextFunction) {
    try {
      const { id } = req.params;
      const user = await this.usersService.updateUser(Number(id), req.body);
      
      res.json({
        success: true,
        data: user
      });
    } catch (error) {
      next(error);
    }
  }

  async deleteUser(req: Request, res: Response, next: NextFunction) {
    try {
      const { id } = req.params;
      await this.usersService.deleteUser(Number(id));
      
      res.json({
        success: true,
        message: 'User deleted successfully'
      });
    } catch (error) {
      next(error);
    }
  }
}
```

## Service Layer

```typescript
// src/services/users.service.ts
import { UsersRepository } from '../repositories/users.repository';
import { CreateUserDTO, UpdateUserDTO, User } from '../types/dto.types';

export class UsersService {
  constructor(
    private usersRepository: UsersRepository,
    private emailService?: EmailService
  ) {}

  async getAllUsers(limit: number, offset: number): Promise<User[]> {
    return this.usersRepository.findMany({ limit, offset });
  }

  async getUserById(id: number): Promise<User | null> {
    return this.usersRepository.findById(id);
  }

  async createUser(dto: CreateUserDTO): Promise<User> {
    // Business logic: validation, transformation, etc.
    const existingUser = await this.usersRepository.findByEmail(dto.email);
    if (existingUser) {
      throw new Error('Email already exists');
    }

    const user = await this.usersRepository.create(dto);

    // Send welcome email
    if (this.emailService) {
      await this.emailService.sendWelcomeEmail(user.email, user.name);
    }

    return user;
  }

  async updateUser(id: number, dto: UpdateUserDTO): Promise<User> {
    const user = await this.usersRepository.findById(id);
    if (!user) {
      throw new Error('User not found');
    }

    return this.usersRepository.update(id, dto);
  }

  async deleteUser(id: number): Promise<void> {
    const user = await this.usersRepository.findById(id);
    if (!user) {
      throw new Error('User not found');
    }

    await this.usersRepository.delete(id);
  }
}
```

## Repository Layer

```typescript
// src/repositories/base.repository.ts
import { Model } from 'objection';

export abstract class BaseRepository<T extends Model> {
  protected Model: typeof Model;

  constructor(Model: typeof Model) {
    this.Model = Model;
  }

  async findMany(options: { limit: number; offset: number }): Promise<T[]> {
    return this.Model.query()
      .limit(options.limit)
      .offset(options.offset);
  }

  async findById(id: number | string): Promise<T | null> {
    return this.Model.query().findById(id);
  }

  async create(data: Partial<T>): Promise<T> {
    return this.Model.query().insert(data);
  }

  async update(id: number | string, data: Partial<T>): Promise<T> {
    return this.Model.query().patch(data).findById(id);
  }

  async delete(id: number | string): Promise<number> {
    return this.Model.query().deleteById(id);
  }
}

// src/repositories/users.repository.ts
import { User } from '../types/dto.types';
import { BaseRepository } from './base.repository';
import UserModel from '../database/models/User';

export class UsersRepository extends BaseRepository<User> {
  constructor() {
    super(UserModel);
  }

  async findByEmail(email: string): Promise<User | null> {
    return UserModel.query().findOne('email', email);
  }

  async findByRole(role: string): Promise<User[]> {
    return UserModel.query().where('role', role);
  }
}
```

## Route Layer

```typescript
// src/routes/users.routes.ts
import { Router } from 'express';
import { UsersController } from '../controllers/users.controller';
import { UsersService } from '../services/users.service';
import { UsersRepository } from '../repositories/users.repository';
import { validateCreateUser, validateUpdateUser } from '../validators/users.validator';
import { asyncHandler } from '../utils/helpers';

const router = Router();

// Dependency injection
const usersRepository = new UsersRepository();
const usersService = new UsersService(usersRepository);
const usersController = new UsersController(usersService);

// Routes
router.get('/', asyncHandler((req, res, next) => 
  usersController.getAllUsers(req, res, next)
));

router.get('/:id', asyncHandler((req, res, next) =>
  usersController.getUserById(req, res, next)
));

router.post('/', validateCreateUser, asyncHandler((req, res, next) =>
  usersController.createUser(req, res, next)
));

router.put('/:id', validateUpdateUser, asyncHandler((req, res, next) =>
  usersController.updateUser(req, res, next)
));

router.delete('/:id', asyncHandler((req, res, next) =>
  usersController.deleteUser(req, res, next)
));

export default router;
```

## Middleware Layer

### Validation Middleware

```typescript
// src/middleware/validation.middleware.ts
import { Request, Response, NextFunction } from 'express';
import { body, validationResult } from 'express-validator';

export const validateCreateUser = [
  body('name').isString().trim().notEmpty(),
  body('email').isEmail().normalizeEmail(),
  body('password').isLength({ min: 8 }),
  (req: Request, res: Response, next: NextFunction) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({
        success: false,
        errors: errors.array()
      });
    }
    next();
  }
];
```

### Authentication Middleware

```typescript
// src/middleware/auth.middleware.ts
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';

export interface AuthRequest extends Request {
  user?: { id: string; role: string };
}

export const authMiddleware = (
  req: AuthRequest,
  res: Response,
  next: NextFunction
) => {
  const token = req.headers.authorization?.split(' ')[1];

  if (!token) {
    return res.status(401).json({
      success: false,
      error: 'No token provided'
    });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET!);
    req.user = decoded as { id: string; role: string };
    next();
  } catch (error) {
    return res.status(401).json({
      success: false,
      error: 'Invalid token'
    });
  }
};

export const roleMiddleware = (allowedRoles: string[]) => {
  return (req: AuthRequest, res: Response, next: NextFunction) => {
    if (!req.user || !allowedRoles.includes(req.user.role)) {
      return res.status(403).json({
        success: false,
        error: 'Access denied'
      });
    }
    next();
  };
};
```

### Error Handler Middleware

```typescript
// src/middleware/error-handler.middleware.ts
import { Request, Response, NextFunction } from 'express';

export class AppError extends Error {
  constructor(
    public statusCode: number,
    message: string,
    public code?: string
  ) {
    super(message);
    Error.captureStackTrace(this, this.constructor);
  }
}

export const errorHandler = (
  error: Error | AppError,
  req: Request,
  res: Response,
  next: NextFunction
) => {
  const statusCode = error instanceof AppError ? error.statusCode : 500;
  const code = error instanceof AppError ? error.code : 'INTERNAL_ERROR';

  console.error(`[${code}] ${error.message}`);

  res.status(statusCode).json({
    success: false,
    error: {
      message: error.message,
      code,
      statusCode,
      ...(process.env.NODE_ENV === 'development' && { stack: error.stack })
    }
  });
};
```

## Application Setup

```typescript
// src/app.ts
import express, { Application } from 'express';
import cors from 'cors';
import helmet from 'helmet';
import usersRoutes from './routes/users.routes';
import postsRoutes from './routes/posts.routes';
import authRoutes from './routes/auth.routes';
import { errorHandler } from './middleware/error-handler.middleware';
import { loggingMiddleware } from './middleware/logging.middleware';

export function createApp(): Application {
  const app = express();

  // Middleware
  app.use(helmet());
  app.use(cors());
  app.use(express.json());
  app.use(express.urlencoded({ extended: true }));
  app.use(loggingMiddleware);

  // Routes
  app.use('/api/auth', authRoutes);
  app.use('/api/users', usersRoutes);
  app.use('/api/posts', postsRoutes);

  // Health check
  app.get('/health', (req, res) => {
    res.json({ status: 'OK', timestamp: new Date() });
  });

  // 404 handler
  app.use((req, res) => {
    res.status(404).json({
      success: false,
      error: 'Route not found'
    });
  });

  // Error handler (must be last)
  app.use(errorHandler);

  return app;
}

// src/server.ts
import { createApp } from './app';
import { db } from './config/database.config';

const app = createApp();
const PORT = process.env.PORT || 3000;

// Initialize database
db.initialize().then(() => {
  app.listen(PORT, () => {
    console.log(`Server running on http://localhost:${PORT}`);
  });
}).catch((error) => {
  console.error('Database initialization error:', error);
  process.exit(1);
});
```

## Best Practices

- **Separate Concerns:** Controllers, services, repositories
- **Error Handling:** Centralized error handler middleware
- **Async/Await:** Use async/await with error handling wrapper
- **Validation:** Validate at HTTP boundary
- **Logging:** Log requests and errors
- **Configuration:** Use environment variables
- **Type Safety:** Use TypeScript for compile-time checking
- **Testing:** Test services independently
- **Security:** Use helmet, cors, rate limiting
- **Documentation:** API documentation with Swagger

## File Naming Conventions

```
Controllers:        *.controller.ts      (users.controller.ts)
Services:           *.service.ts         (users.service.ts)
Repositories:       *.repository.ts      (users.repository.ts)
Routes:             *.routes.ts          (users.routes.ts)
Validators:         *.validator.ts       (users.validator.ts)
Middleware:         *.middleware.ts      (auth.middleware.ts)
Types/DTOs:         *.types.ts           (dto.types.ts)
Config:             *.config.ts          (app.config.ts)
Tests:              *.test.ts            (users.service.test.ts)
```

## References & Sources

### Official Documentation
- Express.js: https://expressjs.com/
- Express TypeScript Setup: https://expressjs.com/en/resources/middleware.html
- Node.js Best Practices: https://nodejs.org/en/docs/

### Libraries
- **express-validator:** https://express-validator.github.io/
- **helmet:** https://helmetjs.github.io/
- **cors:** https://expressjs.com/en/resources/middleware/cors.html
- **jsonwebtoken:** https://github.com/auth0/node-jsonwebtoken
- **prisma:** https://www.prisma.io/
- **objection.js:** https://vincit.github.io/objection.js/

### Articles & Patterns
- Clean Architecture in Express
- RESTful API Design
- Repository Pattern
- Service Layer Pattern
- Middleware Composition

### Note
This architecture emphasizes scalability and maintainability. Adjust based on your project's complexity and team experience.
