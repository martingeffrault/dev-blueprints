# Express.js 5.x (2025)

> **Last updated**: January 2026
> **Versions covered**: 5.0, 5.1 (default on npm)
> **Key feature**: Native async/await error handling

---

## Philosophy (2025-2026)

Express.js 5 was released in October 2024 after a decade of development, focusing on **stability and security**. Express 5.1 became the default on npm in March 2025.

**Key philosophical shifts:**
- **Native Promise support** - Rejected promises auto-forward to error middleware
- **No more asyncHandler** - Async routes work natively
- **Security hardening** - ReDoS protection via path-to-regexp v8
- **Modern Node.js** - Requires Node.js 18+
- **TypeScript-first** - Improved type definitions
- **Minimal core** - Use middleware for everything else

**LTS Timeline:**
- CURRENT: Available but not latest (minimum 3 months)
- ACTIVE: Tagged latest on npm (minimum 12 months)
- MAINTENANCE: 12 months after new major becomes ACTIVE

---

## TL;DR

- Use Express 5.x for all new projects
- No need for `express-async-errors` or `asyncHandler` wrappers
- Use new route syntax: `{/:optional}` instead of `:optional?`
- Use Zod for validation with `zod-express-middleware`
- Use Helmet for security headers
- Use `express-rate-limit` for rate limiting
- Use strict TypeScript with proper Request/Response types
- Always define a global error handler
- Structure: routes -> controllers -> services -> repositories

---

## Best Practices

### Project Structure (2025)

```
my-express-app/
├── src/
│   ├── index.ts              # App entry point
│   ├── app.ts                # Express app configuration
│   ├── routes/
│   │   ├── index.ts          # Route aggregator
│   │   ├── users.routes.ts   # User routes
│   │   ├── posts.routes.ts   # Post routes
│   │   └── auth.routes.ts    # Auth routes
│   ├── controllers/
│   │   ├── users.controller.ts
│   │   ├── posts.controller.ts
│   │   └── auth.controller.ts
│   ├── services/
│   │   ├── users.service.ts
│   │   ├── posts.service.ts
│   │   └── auth.service.ts
│   ├── middleware/
│   │   ├── auth.middleware.ts
│   │   ├── validate.middleware.ts
│   │   ├── error.middleware.ts
│   │   └── rateLimiter.middleware.ts
│   ├── schemas/              # Zod schemas
│   │   ├── user.schema.ts
│   │   └── auth.schema.ts
│   ├── types/
│   │   └── index.ts
│   ├── utils/
│   │   └── logger.ts
│   └── config/
│       └── index.ts
├── package.json
├── tsconfig.json
└── .env
```

### Basic App Setup (Express 5 + TypeScript)

```typescript
// src/app.ts
import express, { Express, Request, Response, NextFunction } from 'express';
import helmet from 'helmet';
import cors from 'cors';
import compression from 'compression';
import { rateLimit } from 'express-rate-limit';
import { errorHandler } from './middleware/error.middleware';
import { routes } from './routes';

const app: Express = express();

// Security middleware
app.use(helmet());
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') ?? ['http://localhost:3000'],
  credentials: true,
}));

// Rate limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  limit: 100, // 100 requests per window
  standardHeaders: 'draft-7',
  legacyHeaders: false,
  message: { error: 'Too many requests, please try again later.' },
});
app.use(limiter);

// Body parsing
app.use(express.json({ limit: '10kb' }));
app.use(express.urlencoded({ extended: false })); // Note: extended defaults to false in Express 5
app.use(compression());

// Health check
app.get('/health', (req: Request, res: Response) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

// API routes
app.use('/api', routes);

// 404 handler
app.use((req: Request, res: Response) => {
  res.status(404).json({ error: 'Not Found' });
});

// Global error handler (must be last)
app.use(errorHandler);

export { app };
```

```typescript
// src/index.ts
import { app } from './app';

const PORT = parseInt(process.env.PORT ?? '3000', 10);

const server = app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});

// Graceful shutdown
process.on('SIGTERM', () => {
  console.log('SIGTERM received, shutting down gracefully');
  server.close(() => {
    console.log('Server closed');
    process.exit(0);
  });
});
```

### Route Definitions

```typescript
// src/routes/index.ts
import { Router } from 'express';
import { usersRoutes } from './users.routes';
import { postsRoutes } from './posts.routes';
import { authRoutes } from './auth.routes';

const router = Router();

router.use('/users', usersRoutes);
router.use('/posts', postsRoutes);
router.use('/auth', authRoutes);

export { router as routes };
```

```typescript
// src/routes/users.routes.ts
import { Router } from 'express';
import { UsersController } from '../controllers/users.controller';
import { validateRequest } from '../middleware/validate.middleware';
import { authMiddleware } from '../middleware/auth.middleware';
import { createUserSchema, updateUserSchema, querySchema } from '../schemas/user.schema';

const router = Router();
const controller = new UsersController();

// GET /api/users
router.get(
  '/',
  validateRequest({ query: querySchema }),
  controller.getAll
);

// GET /api/users/:id
router.get('/:id', controller.getById);

// POST /api/users
router.post(
  '/',
  validateRequest({ body: createUserSchema }),
  controller.create
);

// PUT /api/users/:id (protected)
router.put(
  '/:id',
  authMiddleware,
  validateRequest({ body: updateUserSchema }),
  controller.update
);

// DELETE /api/users/:id (protected)
router.delete('/:id', authMiddleware, controller.delete);

export { router as usersRoutes };
```

### Express 5 Route Syntax Changes

```typescript
// Express 5 updated to path-to-regexp v8
// Several syntax changes for security (ReDoS protection)

// Optional parameters
// ❌ Express 4: /users/:id?
// ✅ Express 5: /users{/:id}
router.get('/users{/:id}', handler);

// Optional prefix
// ❌ Express 4: /users/:id/posts?
// ✅ Express 5: /users/:id{/posts}
router.get('/users/:id{/posts}', handler);

// Zero or more (wildcard)
// ❌ Express 4: /files/*
// ✅ Express 5: /files/*path or /files/(*)
router.get('/files/*path', (req, res) => {
  const filePath = req.params.path; // Captured wildcard
  res.send(`File: ${filePath}`);
});

// Regex patterns removed for security
// ❌ No longer supported: /user/:id(\\d+)
// ✅ Validate in middleware instead
router.get('/user/:id', validateNumericId, handler);
```

### Controller Pattern

```typescript
// src/controllers/users.controller.ts
import { Request, Response } from 'express';
import { UsersService } from '../services/users.service';
import { AppError } from '../utils/errors';

export class UsersController {
  private usersService = new UsersService();

  // Express 5: No try-catch needed! Errors auto-forward to error middleware
  getAll = async (req: Request, res: Response) => {
    const { page = 1, limit = 10 } = req.query;
    const result = await this.usersService.findAll({
      page: Number(page),
      limit: Number(limit),
    });

    res.json({
      data: result.users,
      pagination: {
        page: Number(page),
        limit: Number(limit),
        total: result.total,
      },
    });
  };

  getById = async (req: Request, res: Response) => {
    const user = await this.usersService.findById(req.params.id);

    if (!user) {
      throw new AppError('User not found', 404);
    }

    res.json(user);
  };

  create = async (req: Request, res: Response) => {
    const user = await this.usersService.create(req.body);
    res.status(201).json(user);
  };

  update = async (req: Request, res: Response) => {
    const user = await this.usersService.update(req.params.id, req.body);

    if (!user) {
      throw new AppError('User not found', 404);
    }

    res.json(user);
  };

  delete = async (req: Request, res: Response) => {
    await this.usersService.delete(req.params.id);
    res.status(204).send();
  };
}
```

### Validation with Zod

```typescript
// src/schemas/user.schema.ts
import { z } from 'zod';

export const createUserSchema = z.object({
  name: z.string().min(2).max(100),
  email: z.string().email(),
  password: z.string().min(8).max(100),
  role: z.enum(['user', 'admin']).default('user'),
});

export const updateUserSchema = z.object({
  name: z.string().min(2).max(100).optional(),
  email: z.string().email().optional(),
});

export const querySchema = z.object({
  page: z.coerce.number().int().positive().default(1),
  limit: z.coerce.number().int().positive().max(100).default(10),
  search: z.string().optional(),
});

// Infer types from schemas
export type CreateUserInput = z.infer<typeof createUserSchema>;
export type UpdateUserInput = z.infer<typeof updateUserSchema>;
export type QueryInput = z.infer<typeof querySchema>;
```

```typescript
// src/middleware/validate.middleware.ts
import { Request, Response, NextFunction } from 'express';
import { AnyZodObject, ZodError } from 'zod';

interface ValidationSchemas {
  body?: AnyZodObject;
  query?: AnyZodObject;
  params?: AnyZodObject;
}

export function validateRequest(schemas: ValidationSchemas) {
  return async (req: Request, res: Response, next: NextFunction) => {
    try {
      if (schemas.body) {
        req.body = await schemas.body.parseAsync(req.body);
      }
      if (schemas.query) {
        req.query = await schemas.query.parseAsync(req.query);
      }
      if (schemas.params) {
        req.params = await schemas.params.parseAsync(req.params);
      }
      next();
    } catch (error) {
      if (error instanceof ZodError) {
        res.status(400).json({
          error: 'Validation failed',
          details: error.errors.map((e) => ({
            path: e.path.join('.'),
            message: e.message,
          })),
        });
        return;
      }
      next(error);
    }
  };
}
```

### Error Handling (Express 5 Native)

```typescript
// src/utils/errors.ts
export class AppError extends Error {
  constructor(
    public message: string,
    public statusCode: number = 500,
    public isOperational: boolean = true
  ) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
}

export class NotFoundError extends AppError {
  constructor(message = 'Resource not found') {
    super(message, 404);
  }
}

export class UnauthorizedError extends AppError {
  constructor(message = 'Unauthorized') {
    super(message, 401);
  }
}

export class ForbiddenError extends AppError {
  constructor(message = 'Forbidden') {
    super(message, 403);
  }
}

export class ValidationError extends AppError {
  constructor(message = 'Validation failed') {
    super(message, 400);
  }
}
```

```typescript
// src/middleware/error.middleware.ts
import { Request, Response, NextFunction } from 'express';
import { AppError } from '../utils/errors';

export function errorHandler(
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction
) {
  // Log error
  console.error(`[Error] ${err.message}`, {
    stack: err.stack,
    path: req.path,
    method: req.method,
  });

  // Handle known operational errors
  if (err instanceof AppError) {
    res.status(err.statusCode).json({
      error: err.message,
      ...(process.env.NODE_ENV === 'development' && { stack: err.stack }),
    });
    return;
  }

  // Handle Zod validation errors
  if (err.name === 'ZodError') {
    res.status(400).json({
      error: 'Validation failed',
      details: (err as any).errors,
    });
    return;
  }

  // Handle unknown errors
  res.status(500).json({
    error: process.env.NODE_ENV === 'production'
      ? 'Internal server error'
      : err.message,
    ...(process.env.NODE_ENV === 'development' && { stack: err.stack }),
  });
}
```

### Authentication with JWT

```typescript
// src/middleware/auth.middleware.ts
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';
import { UnauthorizedError } from '../utils/errors';

interface JwtPayload {
  userId: string;
  email: string;
  role: string;
}

// Extend Express Request type
declare global {
  namespace Express {
    interface Request {
      user?: JwtPayload;
    }
  }
}

export async function authMiddleware(
  req: Request,
  res: Response,
  next: NextFunction
) {
  const authHeader = req.headers.authorization;

  if (!authHeader?.startsWith('Bearer ')) {
    throw new UnauthorizedError('No token provided');
  }

  const token = authHeader.slice(7);

  try {
    const payload = jwt.verify(
      token,
      process.env.JWT_SECRET!
    ) as JwtPayload;

    req.user = payload;
    next();
  } catch {
    throw new UnauthorizedError('Invalid token');
  }
}

// Role-based authorization
export function requireRole(...roles: string[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) {
      throw new UnauthorizedError('Not authenticated');
    }

    if (!roles.includes(req.user.role)) {
      throw new UnauthorizedError('Insufficient permissions');
    }

    next();
  };
}
```

```typescript
// src/controllers/auth.controller.ts
import { Request, Response } from 'express';
import jwt from 'jsonwebtoken';
import bcrypt from 'bcrypt';
import { UsersService } from '../services/users.service';
import { AppError, UnauthorizedError } from '../utils/errors';

export class AuthController {
  private usersService = new UsersService();

  login = async (req: Request, res: Response) => {
    const { email, password } = req.body;

    const user = await this.usersService.findByEmail(email);
    if (!user) {
      throw new UnauthorizedError('Invalid credentials');
    }

    const isValid = await bcrypt.compare(password, user.passwordHash);
    if (!isValid) {
      throw new UnauthorizedError('Invalid credentials');
    }

    const token = jwt.sign(
      { userId: user.id, email: user.email, role: user.role },
      process.env.JWT_SECRET!,
      { expiresIn: '24h' }
    );

    const refreshToken = jwt.sign(
      { userId: user.id },
      process.env.JWT_REFRESH_SECRET!,
      { expiresIn: '7d' }
    );

    res.json({
      token,
      refreshToken,
      user: {
        id: user.id,
        email: user.email,
        name: user.name,
        role: user.role,
      },
    });
  };

  register = async (req: Request, res: Response) => {
    const { email, password, name } = req.body;

    const existing = await this.usersService.findByEmail(email);
    if (existing) {
      throw new AppError('Email already registered', 409);
    }

    const passwordHash = await bcrypt.hash(password, 12);
    const user = await this.usersService.create({
      email,
      passwordHash,
      name,
      role: 'user',
    });

    res.status(201).json({
      message: 'User registered successfully',
      user: { id: user.id, email: user.email, name: user.name },
    });
  };
}
```

### Security Best Practices

```typescript
// src/app.ts - Security configuration
import express from 'express';
import helmet from 'helmet';
import cors from 'cors';
import { rateLimit } from 'express-rate-limit';

const app = express();

// Helmet sets various security headers
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      scriptSrc: ["'self'"],
      imgSrc: ["'self'", 'data:', 'https:'],
    },
  },
  crossOriginEmbedderPolicy: true,
  crossOriginOpenerPolicy: true,
  crossOriginResourcePolicy: { policy: 'same-site' },
  hsts: {
    maxAge: 31536000, // 1 year
    includeSubDomains: true,
    preload: true,
  },
}));

// Disable X-Powered-By header
app.disable('x-powered-by');

// CORS configuration
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(','),
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true,
  maxAge: 86400, // 24 hours
}));

// Rate limiting - general
const generalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  limit: 100,
  standardHeaders: 'draft-7',
  legacyHeaders: false,
});
app.use(generalLimiter);

// Rate limiting - strict for auth endpoints
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  limit: 5, // 5 attempts per 15 minutes
  skipSuccessfulRequests: true,
  message: { error: 'Too many attempts, please try again later' },
});
app.use('/api/auth/login', authLimiter);
app.use('/api/auth/register', authLimiter);

// Body size limits
app.use(express.json({ limit: '10kb' }));
app.use(express.urlencoded({ extended: false, limit: '10kb' }));
```

### TypeScript Request Type Extensions

```typescript
// src/types/index.ts
import { Request } from 'express';
import { z } from 'zod';
import { createUserSchema, querySchema } from '../schemas/user.schema';

// Type-safe request with validated body
export interface TypedRequestBody<T> extends Request {
  body: T;
}

// Type-safe request with validated query
export interface TypedRequestQuery<T> extends Request {
  query: T;
}

// Combined typed request
export interface TypedRequest<TBody = any, TQuery = any, TParams = any> extends Request {
  body: TBody;
  query: TQuery;
  params: TParams;
}

// Usage in controller
export type CreateUserRequest = TypedRequestBody<z.infer<typeof createUserSchema>>;
export type GetUsersRequest = TypedRequestQuery<z.infer<typeof querySchema>>;
```

---

## Anti-Patterns

### No asyncHandler Needed in Express 5

**Why it's bad**: Unnecessary wrapper, Express 5 handles this natively.

```typescript
// ❌ DON'T - Express 4 pattern (no longer needed)
import asyncHandler from 'express-async-handler';

router.get('/users', asyncHandler(async (req, res) => {
  const users = await getUsers();
  res.json(users);
}));

// ❌ DON'T - Manual try-catch everywhere
router.get('/users', async (req, res, next) => {
  try {
    const users = await getUsers();
    res.json(users);
  } catch (error) {
    next(error);
  }
});

// ✅ DO - Express 5 native async support
router.get('/users', async (req, res) => {
  const users = await getUsers(); // Errors auto-forward to error middleware
  res.json(users);
});
```

### Using Old Route Syntax

**Why it's bad**: Express 5's path-to-regexp v8 has breaking changes.

```typescript
// ❌ DON'T - Express 4 optional parameter syntax
router.get('/users/:id?', handler);

// ✅ DO - Express 5 optional parameter syntax
router.get('/users{/:id}', handler);

// ❌ DON'T - Regex patterns in routes (security risk)
router.get('/user/:id(\\d+)', handler);

// ✅ DO - Validate in middleware
const validateNumericId = (req, res, next) => {
  if (!/^\d+$/.test(req.params.id)) {
    return res.status(400).json({ error: 'Invalid ID format' });
  }
  next();
};
router.get('/user/:id', validateNumericId, handler);
```

### Missing Global Error Handler

**Why it's bad**: Unhandled errors crash the app or leak sensitive info.

```typescript
// ❌ DON'T - No error handler
const app = express();
app.use('/api', routes);
// Missing error handler!

// ✅ DO - Always add global error handler
const app = express();
app.use('/api', routes);

// Must have 4 parameters to be recognized as error handler
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  console.error(err);
  res.status(500).json({
    error: process.env.NODE_ENV === 'production'
      ? 'Internal server error'
      : err.message,
  });
});
```

### Using req.param() (Removed in Express 5)

**Why it's bad**: Method completely removed.

```typescript
// ❌ DON'T - Removed in Express 5
const id = req.param('id');

// ✅ DO - Use specific sources
const id = req.params.id;      // URL parameters
const page = req.query.page;   // Query string
const name = req.body.name;    // Request body
```

### Not Validating Input

**Why it's bad**: Security vulnerabilities, type issues.

```typescript
// ❌ DON'T - Trust user input
router.post('/users', async (req, res) => {
  const user = await db.users.create(req.body); // Dangerous!
  res.json(user);
});

// ✅ DO - Validate with Zod
router.post(
  '/users',
  validateRequest({ body: createUserSchema }),
  async (req, res) => {
    const user = await db.users.create(req.body); // Validated!
    res.json(user);
  }
);
```

### Business Logic in Routes

**Why it's bad**: Hard to test, violates separation of concerns.

```typescript
// ❌ DON'T - All logic in route handler
router.post('/users', async (req, res) => {
  const existingUser = await db.users.findByEmail(req.body.email);
  if (existingUser) {
    return res.status(409).json({ error: 'Email exists' });
  }
  const hashedPassword = await bcrypt.hash(req.body.password, 12);
  const user = await db.users.create({
    ...req.body,
    password: hashedPassword,
  });
  // Send email, log analytics, etc...
  res.status(201).json(user);
});

// ✅ DO - Separate concerns
// Controller
router.post('/users', validateRequest({ body: createUserSchema }), controller.create);

// Service
class UsersService {
  async create(data: CreateUserInput) {
    const existing = await this.repository.findByEmail(data.email);
    if (existing) throw new AppError('Email exists', 409);

    const hashedPassword = await bcrypt.hash(data.password, 12);
    return this.repository.create({ ...data, password: hashedPassword });
  }
}
```

### Not Using Security Middleware

**Why it's bad**: Vulnerable to common attacks.

```typescript
// ❌ DON'T - No security headers
const app = express();
app.use(express.json());
app.use('/api', routes);

// ✅ DO - Use Helmet and rate limiting
import helmet from 'helmet';
import { rateLimit } from 'express-rate-limit';

const app = express();
app.use(helmet());
app.use(rateLimit({
  windowMs: 15 * 60 * 1000,
  limit: 100,
}));
app.use(express.json({ limit: '10kb' }));
app.use('/api', routes);
```

### Incorrect HTTP Status Codes

**Why it's bad**: Express 5 enforces valid status codes.

```typescript
// ❌ DON'T - Invalid status codes (Express 5 throws error)
res.status(999).send('Error');  // Invalid!
res.status('200').send('OK');   // String not allowed!

// ✅ DO - Use valid HTTP status codes
res.status(200).json({ data });
res.status(201).json({ created: true });
res.status(400).json({ error: 'Bad request' });
res.status(404).json({ error: 'Not found' });
res.status(500).json({ error: 'Server error' });
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 5.0.0 | Oct 2024 | Major release: async/await support, path-to-regexp v8, Node 18+ |
| 5.0.1 | Nov 2024 | Bug fixes, improved TypeScript types |
| 5.1.0 | Mar 2025 | Default on npm, Uint8Array support in `res.send()`, Brotli compression |

### Breaking Changes from Express 4

| Feature | Express 4 | Express 5 |
|---------|-----------|-----------|
| Async error handling | Manual `next(err)` | Automatic |
| Optional params | `:param?` | `{/:param}` |
| Wildcard routes | `*` | `*name` or `(*)` |
| Regex in routes | Supported | Removed (security) |
| `req.param()` | Deprecated | Removed |
| `res.status()` | String/number | Number only (valid HTTP codes) |
| Method pluralization | `req.acceptsCharset()` | `req.acceptsCharsets()` |
| URL-encoded depth | Default unlimited | Default limited |
| `extended` option | Default `true` | Default `false` |
| Node.js version | Node 0.10+ | Node 18+ |

---

## Quick Reference

### Route Methods

| Method | Usage |
|--------|-------|
| `app.get()` | GET requests |
| `app.post()` | POST requests |
| `app.put()` | PUT requests |
| `app.patch()` | PATCH requests |
| `app.delete()` | DELETE requests |
| `app.all()` | All HTTP methods |
| `app.use()` | Middleware mount |
| `app.route()` | Chainable route handlers |

### Request Object

| Property/Method | Description |
|-----------------|-------------|
| `req.params` | URL route parameters |
| `req.query` | Query string parameters |
| `req.body` | Parsed request body |
| `req.headers` | Request headers |
| `req.cookies` | Cookies (with cookie-parser) |
| `req.ip` | Client IP address |
| `req.method` | HTTP method |
| `req.path` | URL path |
| `req.hostname` | Host name |
| `req.get(header)` | Get header value |

### Response Object

| Method | Description |
|--------|-------------|
| `res.json()` | Send JSON response |
| `res.send()` | Send response (auto content-type) |
| `res.status()` | Set HTTP status code |
| `res.redirect()` | Redirect request |
| `res.render()` | Render view template |
| `res.sendFile()` | Send file |
| `res.download()` | Prompt file download |
| `res.cookie()` | Set cookie |
| `res.clearCookie()` | Clear cookie |
| `res.set()` | Set response header |

### Essential Middleware

| Package | Purpose |
|---------|---------|
| `helmet` | Security headers |
| `cors` | CORS handling |
| `express-rate-limit` | Rate limiting |
| `compression` | Response compression |
| `cookie-parser` | Cookie parsing |
| `express-session` | Session management |
| `morgan` | HTTP request logging |

### Dependencies (package.json)

```json
{
  "dependencies": {
    "express": "^5.1.0",
    "helmet": "^8.0.0",
    "cors": "^2.8.5",
    "express-rate-limit": "^7.5.0",
    "compression": "^1.7.5",
    "zod": "^3.24.0",
    "jsonwebtoken": "^9.0.2",
    "bcrypt": "^5.1.1"
  },
  "devDependencies": {
    "@types/express": "^5.0.0",
    "@types/node": "^22.0.0",
    "@types/cors": "^2.8.17",
    "@types/compression": "^1.7.5",
    "@types/jsonwebtoken": "^9.0.7",
    "@types/bcrypt": "^5.0.2",
    "typescript": "^5.7.0"
  }
}
```

---

## Resources

- [Express.js Official Documentation](https://expressjs.com/)
- [Express 5.0 Release Notes](https://expressjs.com/2024/10/15/v5-release.html)
- [Express 5.1 Default Release](https://expressjs.com/2025/03/31/v5-1-latest-release.html)
- [Migrating from Express 4 to 5](https://expressjs.com/en/guide/migrating-5.html)
- [Express Security Best Practices](https://expressjs.com/en/advanced/best-practice-security.html)
- [Helmet.js Documentation](https://helmetjs.github.io/)
- [express-rate-limit Documentation](https://express-rate-limit.mintlify.app/)
- [Zod Documentation](https://zod.dev/)
