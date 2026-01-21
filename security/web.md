# Web Security (2025)

> **Last updated**: January 2025
> **Scope**: OWASP, XSS, CSRF, headers, input validation

---

## Philosophy

Security is not optional. Every input is malicious until proven otherwise. Defense in depth — multiple layers of protection.

---

## OWASP Top 10 (2025)

1. Broken Access Control
2. Cryptographic Failures
3. Injection
4. Insecure Design
5. Security Misconfiguration
6. Vulnerable Components
7. Auth Failures
8. Data Integrity Failures
9. Logging Failures
10. SSRF

---

## Security Headers

```typescript
// next.config.js
const securityHeaders = [
  {
    key: 'X-DNS-Prefetch-Control',
    value: 'on'
  },
  {
    key: 'Strict-Transport-Security',
    value: 'max-age=63072000; includeSubDomains; preload'
  },
  {
    key: 'X-Frame-Options',
    value: 'SAMEORIGIN'
  },
  {
    key: 'X-Content-Type-Options',
    value: 'nosniff'
  },
  {
    key: 'Referrer-Policy',
    value: 'strict-origin-when-cross-origin'
  },
  {
    key: 'Permissions-Policy',
    value: 'camera=(), microphone=(), geolocation=()'
  }
];
```

---

## XSS Prevention

### ❌ DON'T

```typescript
// Never trust user input
<div dangerouslySetInnerHTML={{ __html: userInput }} />
```

### ✅ DO

```typescript
// React auto-escapes by default
<div>{userInput}</div>

// If you MUST render HTML, sanitize first
import DOMPurify from 'dompurify';
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userInput) }} />
```

---

## CSRF Protection

```typescript
// Next.js Server Actions have CSRF protection built-in
// For API routes, use tokens

import { csrf } from '@/lib/csrf';

export async function POST(request: Request) {
  await csrf.verify(request);
  // ...
}
```

---

## Input Validation

```typescript
// Always validate on server
import { z } from 'zod';

const userSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
  age: z.number().int().positive().max(150),
});

// In API route
const result = userSchema.safeParse(body);
if (!result.success) {
  return Response.json({ error: result.error }, { status: 400 });
}
```

---

## SQL Injection

### ❌ DON'T

```typescript
// Never interpolate user input
const query = `SELECT * FROM users WHERE id = '${userId}'`;
```

### ✅ DO

```typescript
// Use parameterized queries
const user = await prisma.user.findUnique({
  where: { id: userId }
});

// Or with raw SQL
const user = await db.$queryRaw`
  SELECT * FROM users WHERE id = ${userId}
`;
```

---

## Quick Reference

| Threat | Prevention |
|--------|------------|
| XSS | Auto-escape, CSP, sanitize |
| CSRF | Tokens, SameSite cookies |
| Injection | Parameterized queries |
| Broken Auth | Strong passwords, MFA |
| Sensitive Data | Encryption, HTTPS |
