# TypeScript 5.x (2025)

> **Last updated**: January 2026
> **Versions covered**: 5.5, 5.6, 5.7, 5.8+
> **Key feature**: Native Node.js support (22.6+)

---

## Philosophy (2025-2026)

TypeScript in 2025-2026 has become even more central to JavaScript development with **native runtime support in Node.js** and continued focus on **stricter type safety**.

**Key philosophical shifts:**
- **Strict mode is mandatory** — No exceptions, ever
- **Native execution** — Node.js 22.6+ runs `.ts` files directly
- **`satisfies`** — Prefer over type assertions for inference
- **`unknown` over `any`** — Always narrow, never escape
- **Runtime validation** — Pair with Zod/Valibot for full safety
- **TypeScript Compiler in Go** — Rewrite in progress for 10x performance

---

## TL;DR

- Always enable `strict: true` in tsconfig.json
- Use `unknown` instead of `any`, then narrow with type guards
- Use `satisfies` to validate types while preserving inference
- Use discriminated unions for variant types
- Use `const` type parameters for literal type preservation
- Use `using` for automatic resource cleanup (5.2+)
- Run TypeScript directly with `node --experimental-strip-types` (22.6+)
- Enable `--erasableSyntaxOnly` for Node.js direct execution (5.8+)

---

## Best Practices

### Strict Mode Configuration

```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "exactOptionalPropertyTypes": true,
    "noPropertyAccessFromIndexSignature": true,
    "forceConsistentCasingInFileNames": true,
    "verbatimModuleSyntax": true,
    "moduleResolution": "bundler",
    "module": "ESNext",
    "target": "ES2024",
    "lib": ["ES2024"]
  }
}
```

### Type Narrowing with Type Guards

```typescript
// ✅ Use type guards for runtime safety
function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'email' in value
  );
}

function processData(data: unknown) {
  if (isUser(data)) {
    // data is now typed as User
    console.log(data.email);
  }
}
```

### `satisfies` Operator

```typescript
// ✅ Validate type while preserving literal inference
const config = {
  port: 3000,
  host: 'localhost',
  features: ['auth', 'api'] as const
} satisfies Config;

// config.port is number (not any)
// config.features is readonly ['auth', 'api'] (not string[])
```

### Discriminated Unions

```typescript
// ✅ Use discriminated unions for variants
type Result<T, E = Error> =
  | { success: true; data: T }
  | { success: false; error: E };

function handleResult<T>(result: Result<T>) {
  if (result.success) {
    return result.data; // TypeScript knows data exists
  }
  throw result.error; // TypeScript knows error exists
}

// ✅ For API responses
type ApiResponse =
  | { status: 'loading' }
  | { status: 'success'; data: User[] }
  | { status: 'error'; message: string };
```

### Const Type Parameters (5.0+)

```typescript
// ✅ Preserve literal types
function createConfig<const T extends readonly string[]>(values: T) {
  return values;
}

const features = createConfig(['auth', 'api', 'admin']);
// Type: readonly ['auth', 'api', 'admin']
// Not: string[]
```

### Using Declarations (5.2+)

```typescript
// ✅ Automatic resource cleanup
async function processFile(path: string) {
  using file = await openFile(path); // Auto-closed when scope exits
  const content = await file.read();
  return content;
} // file.dispose() called automatically

// Works with Symbol.dispose / Symbol.asyncDispose
class DatabaseConnection {
  [Symbol.dispose]() {
    this.close();
  }
}
```

### Template Literal Types

```typescript
// ✅ Type-safe string patterns
type HttpMethod = 'GET' | 'POST' | 'PUT' | 'DELETE';
type ApiRoute = `/api/${string}`;
type Endpoint = `${HttpMethod} ${ApiRoute}`;

const endpoint: Endpoint = 'GET /api/users'; // ✅
const invalid: Endpoint = 'PATCH /api/users'; // ❌ Type error
```

### Runtime Validation with Zod

```typescript
import { z } from 'zod';

// ✅ Define schema once, infer type
const UserSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  role: z.enum(['admin', 'user']),
  createdAt: z.coerce.date()
});

type User = z.infer<typeof UserSchema>;

// Runtime validation
function createUser(input: unknown): User {
  return UserSchema.parse(input); // Throws on invalid
}
```

---

## Anti-Patterns

### ❌ Using `any`

**Why it's bad**: Bypasses all type checking, defeats the purpose of TypeScript.

```typescript
// ❌ DON'T
function processData(data: any) {
  return data.whatever.you.want; // No errors, no safety
}

// ✅ DO — Use unknown and narrow
function processData(data: unknown) {
  if (isValidData(data)) {
    return data.value;
  }
  throw new Error('Invalid data');
}
```

### ❌ Type Assertions Over Type Guards

**Why it's bad**: Assertions lie to the compiler; guards prove types.

```typescript
// ❌ DON'T — Trusting user input
const user = JSON.parse(body) as User; // Dangerous!

// ✅ DO — Validate at runtime
const user = UserSchema.parse(JSON.parse(body));
```

### ❌ Not Handling Null/Undefined

**Why it's bad**: Runtime crashes from unhandled nullish values.

```typescript
// ❌ DON'T
function getUser(id: string) {
  const user = users.find(u => u.id === id);
  return user.name; // 💥 Crashes if not found
}

// ✅ DO
function getUser(id: string) {
  const user = users.find(u => u.id === id);
  if (!user) {
    throw new Error(`User ${id} not found`);
  }
  return user.name;
}

// ✅ OR use optional chaining + nullish coalescing
return user?.name ?? 'Unknown';
```

### ❌ Enum for Simple Values

**Why it's bad**: Enums have runtime overhead; const objects are cleaner.

```typescript
// ❌ DON'T
enum Status {
  Active = 'active',
  Inactive = 'inactive'
}

// ✅ DO — Const object with as const
const Status = {
  Active: 'active',
  Inactive: 'inactive'
} as const;

type Status = typeof Status[keyof typeof Status];
// Type: 'active' | 'inactive'
```

### ❌ Optional Properties for Variants

**Why it's bad**: Allows invalid combinations.

```typescript
// ❌ DON'T
interface Response {
  data?: User[];
  error?: string;
  loading?: boolean;
}
// Can have { data: [...], error: 'fail' } — invalid!

// ✅ DO — Discriminated union
type Response =
  | { status: 'loading' }
  | { status: 'success'; data: User[] }
  | { status: 'error'; error: string };
```

### ❌ Not Using `noUncheckedIndexedAccess`

**Why it's bad**: Array/object access returns `T` not `T | undefined`.

```typescript
// Without noUncheckedIndexedAccess
const arr = [1, 2, 3];
const val = arr[10]; // Type: number (wrong!)

// With noUncheckedIndexedAccess
const val = arr[10]; // Type: number | undefined (correct!)
if (val !== undefined) {
  console.log(val);
}
```

### ❌ Using `==` Instead of `===`

**Why it's bad**: Implicit coercion causes unexpected behavior.

```typescript
// ❌ DON'T
if (value == null) { } // Matches null AND undefined

// ✅ DO — Be explicit
if (value === null || value === undefined) { }
// Or use nullish check:
if (value == null) { } // Actually fine for null/undefined specifically
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 5.5 | Jun 2024 | Inferred type predicates, isolated declarations |
| 5.6 | Sep 2024 | Disallowed nullish/truthy checks improvements |
| 5.7 | Nov 2024 | `--rewriteRelativeImportExtensions`, ES2024 target, faster startup |
| 5.8 | Mar 2025 | `--erasableSyntaxOnly` for Node.js direct execution, `require()` ESM support |
| - | Jul 2025 | Node.js 22.18+ native TypeScript support unflagged |
| - | 2025 | TypeScript compiler rewrite in Go announced (10x performance) |

### TypeScript 5.8 Highlights

**`--erasableSyntaxOnly`**: Ensures code is compatible with Node.js direct execution:
```bash
node --experimental-strip-types app.ts
```
Errors on constructs like `enum` and `namespace` that can't be stripped.

**Better Monorepo Support**: Improved tsconfig.json resolution in nested projects.

---

## Quick Reference

| Task | Solution |
|------|----------|
| Strict safety | `strict: true` + `noUncheckedIndexedAccess` |
| Type from value | `typeof myObject` |
| Keys of object | `keyof typeof myObject` |
| Values as type | `(typeof obj)[keyof typeof obj]` |
| Validate + infer | `satisfies TypeName` |
| Type from array | `typeof arr[number]` |
| Const literal | `as const` |
| Resource cleanup | `using` keyword |
| Runtime validation | Zod, Valibot, ArkType |
| Run TS directly | `node --experimental-strip-types file.ts` |

---

## Recommended tsconfig.json (2025)

```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "compilerOptions": {
    "target": "ES2024",
    "lib": ["ES2024"],
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "resolveJsonModule": true,
    "esModuleInterop": true,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "exactOptionalPropertyTypes": true,
    "forceConsistentCasingInFileNames": true,
    "verbatimModuleSyntax": true,
    "skipLibCheck": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "./dist"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

---

## Resources

- [Official TypeScript Documentation](https://www.typescriptlang.org/docs/)
- [TypeScript 5.8 Release Notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-8.html)
- [TypeScript 5.7 Release Notes](https://devblogs.microsoft.com/typescript/announcing-typescript-5-7/)
- [Effective TypeScript](https://effectivetypescript.com/)
- [Total TypeScript](https://www.totaltypescript.com/)
- [TypeScript Strict Mode Guide](https://betterstack.com/community/guides/scaling-nodejs/typescript-strict-option/)
