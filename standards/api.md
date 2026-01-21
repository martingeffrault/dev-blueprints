# API Design (2025)

> **Last updated**: January 2026
> **Scope**: REST, GraphQL, versioning, security, error handling
> **Purpose**: Consistent, secure, and well-documented APIs

---

## Philosophy

APIs are contracts. Design for the consumer, not the implementation. Be consistent, predictable, and well-documented. In 2025, API vulnerabilities caused 72% of data breaches — security is not optional.

---

## REST Conventions

### URL Structure

```
/<resource>                 # Collection
/<resource>/{id}            # Single item
/<resource>/{id}/<sub>      # Sub-resource
```

**Rules:**
- Use plural nouns: `/users`, not `/user`
- Use lowercase with hyphens: `/user-profiles`, not `/userProfiles`
- Use nouns, not verbs: `/orders`, not `/getOrders`
- Keep nesting shallow (max 2 levels)

### HTTP Methods

| Method | Use | Idempotent | Safe |
|--------|-----|------------|------|
| GET | Read | Yes | Yes |
| POST | Create | No | No |
| PUT | Replace | Yes | No |
| PATCH | Partial update | Yes | No |
| DELETE | Delete | Yes | No |

### Examples

```
GET    /users                    # List users
GET    /users/123                # Get user 123
POST   /users                    # Create user
PUT    /users/123                # Replace user 123
PATCH  /users/123                # Update user 123
DELETE /users/123                # Delete user 123

GET    /users/123/orders         # Get user's orders
POST   /users/123/orders         # Create order for user

# Avoid deep nesting
GET    /orders/456               # Better than /users/123/orders/456
GET    /orders?user_id=123       # Filter by user
```

---

## Response Format

### Success Response

```json
{
  "data": {
    "id": 123,
    "name": "John Doe",
    "email": "john@example.com"
  }
}
```

### Collection Response

```json
{
  "data": [
    { "id": 1, "name": "Item 1" },
    { "id": 2, "name": "Item 2" }
  ],
  "meta": {
    "page": 1,
    "per_page": 20,
    "total": 100,
    "total_pages": 5
  },
  "links": {
    "self": "/items?page=1",
    "next": "/items?page=2",
    "last": "/items?page=5"
  }
}
```

### Error Response

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input data",
    "details": [
      {
        "field": "email",
        "message": "Must be a valid email address"
      },
      {
        "field": "age",
        "message": "Must be a positive number"
      }
    ],
    "request_id": "req_abc123"
  }
}
```

---

## Status Codes

| Code | Name | When to Use |
|------|------|-------------|
| 200 | OK | Success (GET, PATCH, PUT) |
| 201 | Created | Resource created (POST) |
| 204 | No Content | Success, no body (DELETE) |
| 400 | Bad Request | Malformed request |
| 401 | Unauthorized | No/invalid authentication |
| 403 | Forbidden | Authenticated but not authorized |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Duplicate, version conflict |
| 422 | Unprocessable Entity | Validation errors |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server error |
| 503 | Service Unavailable | Temporarily unavailable |

### Implementation

```typescript
// Express/Node.js example
app.post('/users', async (req, res) => {
  try {
    const result = userSchema.safeParse(req.body);

    if (!result.success) {
      return res.status(422).json({
        error: {
          code: 'VALIDATION_ERROR',
          message: 'Invalid input',
          details: result.error.flatten().fieldErrors,
        },
      });
    }

    const existingUser = await db.user.findUnique({
      where: { email: result.data.email },
    });

    if (existingUser) {
      return res.status(409).json({
        error: {
          code: 'CONFLICT',
          message: 'Email already registered',
        },
      });
    }

    const user = await db.user.create({ data: result.data });

    return res.status(201).json({ data: user });
  } catch (error) {
    logger.error('Failed to create user', error);
    return res.status(500).json({
      error: {
        code: 'INTERNAL_ERROR',
        message: 'An error occurred',
        request_id: req.id,
      },
    });
  }
});
```

---

## Versioning

### URL Versioning (Recommended)

```
/api/v1/users
/api/v2/users
```

```typescript
// Next.js App Router
// app/api/v1/users/route.ts
export async function GET() { /* v1 logic */ }

// app/api/v2/users/route.ts
export async function GET() { /* v2 logic */ }
```

### Header Versioning (Alternative)

```
GET /api/users
Accept: application/vnd.myapi.v2+json
```

### When to Version

- Breaking changes (field removal, type changes)
- Significant behavior changes
- NOT for additive changes (new fields, new endpoints)

---

## Pagination

### Offset-Based (Simple)

```
GET /users?page=2&per_page=20
```

```typescript
// Implementation
const page = parseInt(req.query.page) || 1;
const perPage = Math.min(parseInt(req.query.per_page) || 20, 100);
const offset = (page - 1) * perPage;

const [users, total] = await Promise.all([
  db.user.findMany({ skip: offset, take: perPage }),
  db.user.count(),
]);

res.json({
  data: users,
  meta: {
    page,
    per_page: perPage,
    total,
    total_pages: Math.ceil(total / perPage),
  },
});
```

### Cursor-Based (Large Datasets)

```
GET /users?cursor=eyJpZCI6MTIzfQ&limit=20
```

```typescript
// Implementation
const limit = Math.min(parseInt(req.query.limit) || 20, 100);
const cursor = req.query.cursor
  ? JSON.parse(Buffer.from(req.query.cursor, 'base64').toString())
  : null;

const users = await db.user.findMany({
  take: limit + 1,  // Fetch one extra to check if more exist
  ...(cursor && {
    cursor: { id: cursor.id },
    skip: 1,  // Skip cursor item
  }),
  orderBy: { id: 'asc' },
});

const hasMore = users.length > limit;
const data = hasMore ? users.slice(0, -1) : users;
const nextCursor = hasMore
  ? Buffer.from(JSON.stringify({ id: data[data.length - 1].id })).toString('base64')
  : null;

res.json({
  data,
  meta: { has_more: hasMore },
  links: {
    next: nextCursor ? `/users?cursor=${nextCursor}&limit=${limit}` : null,
  },
});
```

---

## Filtering & Sorting

### Filtering

```
GET /users?status=active
GET /users?role=admin&status=active
GET /users?created_at[gte]=2025-01-01
GET /users?name[like]=john
```

```typescript
// Implementation with Zod validation
const filterSchema = z.object({
  status: z.enum(['active', 'inactive']).optional(),
  role: z.string().optional(),
  created_at: z.object({
    gte: z.string().datetime().optional(),
    lte: z.string().datetime().optional(),
  }).optional(),
});

const filters = filterSchema.parse(req.query);
const where = {
  ...(filters.status && { status: filters.status }),
  ...(filters.role && { role: filters.role }),
  ...(filters.created_at && {
    createdAt: {
      ...(filters.created_at.gte && { gte: new Date(filters.created_at.gte) }),
      ...(filters.created_at.lte && { lte: new Date(filters.created_at.lte) }),
    },
  }),
};
```

### Sorting

```
GET /users?sort=name:asc
GET /users?sort=created_at:desc,name:asc
```

```typescript
// Implementation
const sortParam = req.query.sort || 'created_at:desc';
const orderBy = sortParam.split(',').map(s => {
  const [field, direction] = s.split(':');
  return { [field]: direction || 'asc' };
});
```

### Sparse Fields

```
GET /users?fields=id,name,email
```

```typescript
// Implementation
const fields = req.query.fields?.split(',') || null;
const select = fields?.reduce((acc, f) => ({ ...acc, [f]: true }), {});

const users = await db.user.findMany({ select });
```

---

## Rate Limiting

```typescript
// Headers to include
res.setHeader('X-RateLimit-Limit', '100');
res.setHeader('X-RateLimit-Remaining', '95');
res.setHeader('X-RateLimit-Reset', '1706745600');

// When limit exceeded
res.status(429).json({
  error: {
    code: 'RATE_LIMIT_EXCEEDED',
    message: 'Too many requests',
    retry_after: 60,
  },
});
res.setHeader('Retry-After', '60');
```

### Implementation

```typescript
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(100, '1 m'),
});

app.use(async (req, res, next) => {
  const identifier = req.user?.id || req.ip;
  const { success, limit, remaining, reset } = await ratelimit.limit(identifier);

  res.setHeader('X-RateLimit-Limit', limit);
  res.setHeader('X-RateLimit-Remaining', remaining);
  res.setHeader('X-RateLimit-Reset', reset);

  if (!success) {
    return res.status(429).json({
      error: {
        code: 'RATE_LIMIT_EXCEEDED',
        message: 'Too many requests',
      },
    });
  }

  next();
});
```

---

## Security Headers

```typescript
// Required headers for API responses
app.use((req, res, next) => {
  // Prevent MIME sniffing
  res.setHeader('X-Content-Type-Options', 'nosniff');

  // No caching for API responses (adjust per endpoint)
  res.setHeader('Cache-Control', 'no-store');

  // Request ID for tracing
  res.setHeader('X-Request-ID', req.id);

  next();
});

// CORS configuration
app.use(cors({
  origin: ['https://app.example.com'],
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  exposedHeaders: ['X-RateLimit-Limit', 'X-RateLimit-Remaining'],
  credentials: true,
  maxAge: 86400,
}));
```

---

## Authentication

```typescript
// Bearer token in Authorization header
// Authorization: Bearer <token>

app.use('/api', async (req, res, next) => {
  const authHeader = req.headers.authorization;

  if (!authHeader?.startsWith('Bearer ')) {
    return res.status(401).json({
      error: {
        code: 'UNAUTHORIZED',
        message: 'Missing or invalid authorization header',
      },
    });
  }

  const token = authHeader.slice(7);

  try {
    const payload = await verifyToken(token);
    req.user = payload;
    next();
  } catch {
    return res.status(401).json({
      error: {
        code: 'UNAUTHORIZED',
        message: 'Invalid or expired token',
      },
    });
  }
});
```

---

## Request Tracing

```typescript
import { v4 as uuidv4 } from 'uuid';

// Generate or propagate request ID
app.use((req, res, next) => {
  req.id = req.headers['x-request-id'] || uuidv4();
  res.setHeader('X-Request-ID', req.id);
  next();
});

// Include in logs
logger.info('Request processed', {
  request_id: req.id,
  method: req.method,
  path: req.path,
  status: res.statusCode,
  duration_ms: Date.now() - req.startTime,
});
```

---

## OpenAPI Documentation

```yaml
# openapi.yaml
openapi: 3.1.0
info:
  title: My API
  version: 1.0.0
  description: API documentation

servers:
  - url: https://api.example.com/v1

paths:
  /users:
    get:
      summary: List users
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            default: 1
        - name: per_page
          in: query
          schema:
            type: integer
            default: 20
            maximum: 100
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserListResponse'
        '401':
          $ref: '#/components/responses/Unauthorized'

components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: integer
        name:
          type: string
        email:
          type: string
          format: email

  responses:
    Unauthorized:
      description: Authentication required
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'

  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

security:
  - bearerAuth: []
```

---

## Quick Reference

| Pattern | Example |
|---------|---------|
| List | `GET /resources` |
| Get one | `GET /resources/{id}` |
| Create | `POST /resources` |
| Replace | `PUT /resources/{id}` |
| Update | `PATCH /resources/{id}` |
| Delete | `DELETE /resources/{id}` |
| Sub-resource | `GET /resources/{id}/items` |
| Search | `GET /resources?q=term` |
| Filter | `GET /resources?status=active` |
| Sort | `GET /resources?sort=name:asc` |
| Paginate | `GET /resources?page=1&per_page=20` |
| Fields | `GET /resources?fields=id,name` |

| Header | Purpose |
|--------|---------|
| `Authorization` | Bearer token |
| `Content-Type` | application/json |
| `Accept` | application/json |
| `X-Request-ID` | Request tracing |
| `X-RateLimit-*` | Rate limit info |
| `Retry-After` | Rate limit reset |

---

## Resources

- [Microsoft API Design Best Practices](https://learn.microsoft.com/en-us/azure/architecture/best-practices/api-design)
- [OpenAPI Specification](https://swagger.io/specification/)
- [JSON:API Specification](https://jsonapi.org/)
- [REST API Best Practices 2025](https://hevodata.com/learn/rest-api-best-practices/)
- [OWASP API Security](https://owasp.org/www-project-api-security/)
