# Zod (2025)

> **Last updated**: January 2026
> **Versions covered**: 3.x, 4.x (beta)
> **Purpose**: TypeScript-first schema validation with static type inference

---

## Philosophy (2025-2026)

Zod is the **standard for runtime validation** in TypeScript, offering schema definitions that double as type declarations.

**Key philosophical shifts:**
- **Schema = Type + Validation** — One source of truth
- **TypeScript-first** — Designed for TS, not retrofitted
- **safeParse over parse** — Never throw, always return results
- **Composable schemas** — Build complex from simple
- **Zero dependencies** — Lightweight and portable
- **Coercion built-in** — Handle dirty data elegantly

---

## TL;DR

- Use `z.infer<typeof schema>` for type extraction
- Use `safeParse()` instead of `parse()` for error handling
- Never use `any` — defeats the purpose of Zod
- Use `refine()` for custom validation logic
- Use `transform()` to modify data during validation
- Use `partial()`, `pick()`, `omit()` to reuse schemas
- Use `.catch()` for fallback values
- Enable `strict: true` in TypeScript config

---

## Best Practices

### Basic Schema Definition

```typescript
import { z } from 'zod';

// Define schema
const UserSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  name: z.string().min(2).max(100),
  age: z.number().int().positive().optional(),
  role: z.enum(['admin', 'user', 'guest']),
  createdAt: z.date(),
});

// Infer TypeScript type from schema
type User = z.infer<typeof UserSchema>;

// Now User is:
// {
//   id: string;
//   email: string;
//   name: string;
//   age?: number | undefined;
//   role: 'admin' | 'user' | 'guest';
//   createdAt: Date;
// }
```

### Safe Parsing (Recommended)

```typescript
// ✅ BEST — Use safeParse for error handling
const result = UserSchema.safeParse(data);

if (!result.success) {
  // Handle validation errors
  console.error(result.error.issues);
  return { error: 'Validation failed', details: result.error.flatten() };
}

// result.data is fully typed as User
const user = result.data;

// ❌ AVOID — parse throws on failure
try {
  const user = UserSchema.parse(data); // Throws ZodError
} catch (error) {
  // Error handling in catch blocks is less type-safe
}
```

### Schema Composition

```typescript
// Base schema
const BaseEntitySchema = z.object({
  id: z.string().uuid(),
  createdAt: z.date(),
  updatedAt: z.date(),
});

// Extend for specific entities
const ProductSchema = BaseEntitySchema.extend({
  name: z.string(),
  price: z.number().positive(),
  category: z.string(),
});

// Create variants with partial/pick/omit
const CreateProductSchema = ProductSchema.omit({
  id: true,
  createdAt: true,
  updatedAt: true
});

const UpdateProductSchema = CreateProductSchema.partial();

const ProductSummarySchema = ProductSchema.pick({
  id: true,
  name: true,
  price: true,
});

// Types are automatically inferred
type CreateProduct = z.infer<typeof CreateProductSchema>;
type UpdateProduct = z.infer<typeof UpdateProductSchema>;
type ProductSummary = z.infer<typeof ProductSummarySchema>;
```

### Custom Validation with Refine

```typescript
const PasswordSchema = z
  .string()
  .min(8, 'Password must be at least 8 characters')
  .refine(
    (val) => /[A-Z]/.test(val),
    'Password must contain an uppercase letter'
  )
  .refine(
    (val) => /[0-9]/.test(val),
    'Password must contain a number'
  )
  .refine(
    (val) => /[!@#$%^&*]/.test(val),
    'Password must contain a special character'
  );

// Cross-field validation with superRefine
const SignupSchema = z
  .object({
    password: z.string().min(8),
    confirmPassword: z.string(),
  })
  .superRefine((data, ctx) => {
    if (data.password !== data.confirmPassword) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        message: 'Passwords do not match',
        path: ['confirmPassword'],
      });
    }
  });
```

### Transform Data

```typescript
// Transform during validation
const UserInputSchema = z.object({
  email: z.string().email().toLowerCase(), // Auto-lowercase
  name: z.string().trim(), // Auto-trim whitespace
  birthYear: z.string().transform((val) => parseInt(val, 10)),
});

// Coerce types from strings (great for query params)
const QueryParamsSchema = z.object({
  page: z.coerce.number().int().positive().default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
  includeDeleted: z.coerce.boolean().default(false),
});

// Parse URL query params
const params = QueryParamsSchema.parse({
  page: '2',      // becomes 2
  limit: '50',    // becomes 50
  // includeDeleted defaults to false
});
```

### Default Values and Fallbacks

```typescript
// Default values
const ConfigSchema = z.object({
  host: z.string().default('localhost'),
  port: z.number().default(3000),
  debug: z.boolean().default(false),
});

// Fallback on validation failure
const SafeNumberSchema = z.coerce.number().catch(0);
SafeNumberSchema.parse('invalid'); // Returns 0 instead of throwing

// Nullable vs Optional
const Schema = z.object({
  required: z.string(),                    // string
  optional: z.string().optional(),         // string | undefined
  nullable: z.string().nullable(),         // string | null
  nullish: z.string().nullish(),          // string | null | undefined
});
```

### API Validation Pattern

```typescript
// schemas/api.ts
export const CreateUserRequestSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
  name: z.string().min(2),
});

export const ApiResponseSchema = <T extends z.ZodType>(dataSchema: T) =>
  z.object({
    success: z.boolean(),
    data: dataSchema.optional(),
    error: z.string().optional(),
  });

// Usage in API route (Next.js example)
import { CreateUserRequestSchema } from '@/schemas/api';

export async function POST(request: Request) {
  const body = await request.json();
  const result = CreateUserRequestSchema.safeParse(body);

  if (!result.success) {
    return Response.json(
      { success: false, error: result.error.flatten() },
      { status: 400 }
    );
  }

  const { email, password, name } = result.data;
  // Proceed with validated, typed data
}
```

### Form Validation with React Hook Form

```typescript
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const FormSchema = z.object({
  email: z.string().email('Invalid email address'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
});

type FormData = z.infer<typeof FormSchema>;

function LoginForm() {
  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<FormData>({
    resolver: zodResolver(FormSchema),
  });

  const onSubmit = (data: FormData) => {
    // data is fully typed and validated
    console.log(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('email')} />
      {errors.email && <span>{errors.email.message}</span>}

      <input type="password" {...register('password')} />
      {errors.password && <span>{errors.password.message}</span>}

      <button type="submit">Login</button>
    </form>
  );
}
```

### Environment Variables Validation

```typescript
// env.ts
import { z } from 'zod';

const EnvSchema = z.object({
  NODE_ENV: z.enum(['development', 'production', 'test']),
  DATABASE_URL: z.string().url(),
  API_SECRET: z.string().min(32),
  PORT: z.coerce.number().default(3000),
  DEBUG: z.coerce.boolean().default(false),
});

// Validate at startup
const env = EnvSchema.parse(process.env);

// Export typed env
export { env };

// Usage
import { env } from './env';
console.log(env.DATABASE_URL); // Fully typed
```

---

## Anti-Patterns

### ❌ Using `any` Type

**Why it's bad**: Defeats the purpose of Zod's type inference.

```typescript
// ❌ DON'T
const data: any = await fetchData();
const user = UserSchema.parse(data); // Lost type safety

// ✅ DO
const data: unknown = await fetchData();
const result = UserSchema.safeParse(data);
if (result.success) {
  const user = result.data; // Fully typed
}
```

### ❌ Using parse() Without Error Handling

**Why it's bad**: Unhandled exceptions crash your app.

```typescript
// ❌ DON'T
const user = UserSchema.parse(untrustedData); // Might throw!

// ✅ DO
const result = UserSchema.safeParse(untrustedData);
if (!result.success) {
  // Handle validation error gracefully
  return { error: result.error.flatten() };
}
const user = result.data;
```

### ❌ Duplicating Types and Schemas

**Why it's bad**: Types drift out of sync with validation.

```typescript
// ❌ DON'T — Separate interface and schema
interface User {
  id: string;
  email: string;
  name: string;
}

const UserSchema = z.object({
  id: z.string(),
  email: z.string(),
  name: z.string(),
  age: z.number(), // Forgot to add to interface!
});

// ✅ DO — Single source of truth
const UserSchema = z.object({
  id: z.string(),
  email: z.string(),
  name: z.string(),
  age: z.number(),
});

type User = z.infer<typeof UserSchema>;
```

### ❌ Over-Validating Internal Data

**Why it's bad**: Unnecessary runtime cost.

```typescript
// ❌ DON'T — Validate data you already control
function processUser(user: User) {
  const validated = UserSchema.parse(user); // Redundant!
  return validated;
}

// ✅ DO — Validate at boundaries only
// API endpoints, form submissions, external data
const result = UserSchema.safeParse(externalInput);
```

### ❌ Ignoring Error Details

**Why it's bad**: Users get unhelpful error messages.

```typescript
// ❌ DON'T
if (!result.success) {
  return { error: 'Validation failed' }; // Unhelpful
}

// ✅ DO — Use flatten() for structured errors
if (!result.success) {
  return {
    error: 'Validation failed',
    fieldErrors: result.error.flatten().fieldErrors,
  };
}
// Returns: { email: ['Invalid email'], password: ['Too short'] }
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| **4.0** | Jul 2025 | 14x faster parsing, 57% smaller, Zod Mini, string format top-level |
| 3.24 | 2024 | Last v3 release |

### Zod 4 Breaking Changes

**Performance:**
- **14x** faster string parsing
- **7x** faster array parsing
- **6.5x** faster object parsing
- **57% smaller** bundle size

**String Format Validators → Top-Level Functions:**
```typescript
// ❌ OLD — Method chaining
z.string().email()
z.string().uuid()
z.string().url()
z.string().datetime()
z.string().ip()

// ✅ NEW — Top-level functions
z.email()
z.uuid()     // Strict RFC 9562/4122 validation (variant bits = 10)
z.guid()     // Permissive UUID-like (use if z.uuid() too strict)
z.url()
z.iso.datetime()
z.ipv4()     // Replaces ip() for IPv4
z.ipv6()     // Replaces ip() for IPv6
```

**UUID Validation Stricter:**
```typescript
// ❌ OLD — Permissive
z.string().uuid().parse('00000000-0000-0000-0000-000000000000'); // OK

// ✅ NEW — RFC compliant (variant bits must be 10)
z.uuid().parse('00000000-0000-0000-0000-000000000000'); // Error!
z.guid().parse('00000000-0000-0000-0000-000000000000'); // OK (permissive)
```

**IP Address Validation:**
```typescript
// ❌ OLD — Combined ip()
z.string().ip()

// ✅ NEW — Separate methods, uses new URL() internally (more robust)
z.ipv4()
z.ipv6()
z.union([z.ipv4(), z.ipv6()]) // For both
```

**Unified Error Parameter:**
```typescript
// ❌ OLD — message/errorMap fragmented
z.string().min(5, { message: 'Too short' });
z.string({ errorMap: (issue, ctx) => ({ message: '...' }) });

// ✅ NEW — Unified error param
z.string().min(5, { error: 'Too short' });
z.string().min(5, { error: (issue) => `Min length is ${issue.minimum}` });
z.string({ error: 'Must be a string' });
```

**Refinements Now Internal:**
```typescript
// Internal change: refinements live inside schemas as "checks"
// External API mostly unchanged, but affects custom schema creation
```

### Zod 4 New Features

**@zod/mini (~1.9 KB gzipped):**
```typescript
// NEW — Tree-shakable validation for frontend
import { string, email, object, parse } from '@zod/mini';

// Standalone functions instead of method chaining
const schema = object({
  email: email(),
  name: string(),
});

const result = parse(schema, data); // Tree-shakes unused validators
```

**z.stringbool() for Form Inputs:**
```typescript
// NEW — Converts string booleans to actual booleans
const schema = z.object({
  rememberMe: z.stringbool(), // "true", "false", "1", "0" → boolean
});

schema.parse({ rememberMe: 'true' }); // { rememberMe: true }
schema.parse({ rememberMe: '0' });    // { rememberMe: false }
```

**z.interface() for Recursive Types:**
```typescript
// NEW — Better recursive type handling
const Category = z.interface({
  name: z.string(),
  children: z.lazy(() => z.array(Category)),
});
```

**Locales API:**
```typescript
// NEW — Global error message translation
import { z, setLocale } from 'zod';

setLocale({
  string: {
    email: 'Email invalide',
    min: ({ minimum }) => `Minimum ${minimum} caractères`,
  },
});
```

**z.prettifyError():**
```typescript
// NEW — User-friendly error formatting
const result = schema.safeParse(data);
if (!result.success) {
  console.log(z.prettifyError(result.error));
  // "email: Invalid email address
  //  password: String must contain at least 8 characters"
}
```

### Migration Codemod

```bash
# Automate common migration tasks
npx zod-v3-to-v4 src

# Libraries supporting both v3 and v4 simultaneously:
# - Hono
# - LangChain
# - React Hook Form
```

### Zod Mini vs Zod

| Aspect | Zod | @zod/mini |
|--------|-----|-----------|
| Bundle | ~12 KB | ~1.9 KB |
| Style | Method chaining | Standalone functions |
| Tree-shaking | Partial | Full |
| Use case | Full validation | Frontend-critical |

---

## Quick Reference

| Task | Solution |
|------|----------|
| Define object | `z.object({ key: z.string() })` |
| Infer type | `type T = z.infer<typeof Schema>` |
| Safe parse | `Schema.safeParse(data)` |
| Optional field | `z.string().optional()` |
| Nullable field | `z.string().nullable()` |
| Default value | `z.string().default('value')` |
| Array | `z.array(z.string())` |
| Enum | `z.enum(['a', 'b', 'c'])` |
| Union | `z.union([z.string(), z.number()])` |
| Transform | `z.string().transform(v => v.toUpperCase())` |
| Refine | `z.string().refine(v => v.length > 0)` |
| Extend object | `Schema.extend({ newField: z.string() })` |
| Omit fields | `Schema.omit({ field: true })` |
| Pick fields | `Schema.pick({ field: true })` |
| Partial | `Schema.partial()` |
| Coerce | `z.coerce.number()` |

---

## Integration Examples

| Library | Integration |
|---------|-------------|
| React Hook Form | `zodResolver(Schema)` |
| tRPC | `.input(Schema)` |
| Hono | `zValidator('json', Schema)` |
| Next.js Server Actions | Manual `safeParse()` |
| Express | Middleware with `safeParse()` |

---

## Resources

- [Official Zod Documentation](https://zod.dev/)
- [Zod GitHub Repository](https://github.com/colinhacks/zod)
- [9 Best Practices for Using Zod in 2025](https://javascript.plainenglish.io/9-best-practices-for-using-zod-in-2025-31ee7418062e)
- [Schema Validation with Zod in 2025](https://www.turing.com/blog/data-integrity-through-zod-validation)
- [Better Stack Zod Guide](https://betterstack.com/community/guides/scaling-nodejs/zod-explained/)
- [zod-validation-error Package](https://github.com/causaly/zod-validation-error/)
