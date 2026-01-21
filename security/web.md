# Web Security (2025)

> **Last updated**: January 2026
> **Scope**: OWASP Top 10, XSS, CSRF, CSP, input validation, headers
> **Purpose**: Defense-in-depth security for web applications

---

## Philosophy

Security is not optional. Every input is malicious until proven otherwise. Defense in depth — multiple layers of protection. In 2025, 72% of data breaches originated from API vulnerabilities.

---

## OWASP Top 10:2025

Released November 2025, based on 175,000+ CVEs and global security practitioner feedback.

| # | Category | Key Points |
|---|----------|------------|
| A01 | **Broken Access Control** | #1 risk, SSRF now included, highest CWE count |
| A02 | **Security Misconfiguration** | Jumped from #5 to #2, affects 3% of apps |
| A03 | **Software Supply Chain Failures** | NEW — Dependencies, build systems, distribution |
| A04 | **Cryptographic Failures** | Dropped from #2, still critical |
| A05 | **Injection** | XSS, SQL injection, command injection |
| A06 | **Insecure Design** | Threat modeling improvements observed |
| A07 | **Authentication Failures** | Renamed for clarity, 36 CWEs |
| A08 | **Data Integrity Failures** | Software updates, CI/CD integrity |
| A09 | **Logging & Monitoring Failures** | Detection and response gaps |
| A10 | **Mishandling Exceptional Conditions** | NEW — Error handling, failing open |

### Key Changes from 2021

1. **Supply Chain Failures** (A03) — New category with highest exploit/impact scores
2. **Mishandling Exceptional Conditions** (A10) — 24 CWEs for error handling issues
3. **SSRF consolidated** into Broken Access Control (A01)
4. **Security Misconfiguration** surged to #2

---

## Security Headers (2025)

```typescript
// next.config.js
const securityHeaders = [
  // HTTPS enforcement
  {
    key: 'Strict-Transport-Security',
    value: 'max-age=63072000; includeSubDomains; preload'
  },
  // Clickjacking protection
  {
    key: 'X-Frame-Options',
    value: 'DENY'  // or SAMEORIGIN if needed
  },
  // MIME sniffing protection
  {
    key: 'X-Content-Type-Options',
    value: 'nosniff'
  },
  // Referrer policy
  {
    key: 'Referrer-Policy',
    value: 'strict-origin-when-cross-origin'
  },
  // Feature permissions
  {
    key: 'Permissions-Policy',
    value: 'camera=(), microphone=(), geolocation=(), interest-cohort=()'
  },
  // XSS protection (legacy browsers)
  {
    key: 'X-XSS-Protection',
    value: '1; mode=block'
  },
  // DNS prefetch control
  {
    key: 'X-DNS-Prefetch-Control',
    value: 'on'
  }
];

module.exports = {
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: securityHeaders,
      },
    ];
  },
};
```

### Content Security Policy (CSP)

```typescript
// Strict CSP for production
const cspHeader = `
  default-src 'self';
  script-src 'self' 'strict-dynamic' 'nonce-${nonce}';
  style-src 'self' 'unsafe-inline';
  img-src 'self' blob: data: https:;
  font-src 'self';
  connect-src 'self' https://api.example.com;
  frame-ancestors 'none';
  base-uri 'self';
  form-action 'self';
  upgrade-insecure-requests;
`.replace(/\s{2,}/g, ' ').trim();

// For Next.js middleware
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const nonce = Buffer.from(crypto.randomUUID()).toString('base64');
  const response = NextResponse.next();

  response.headers.set('Content-Security-Policy', cspHeader);
  response.headers.set('X-Nonce', nonce);

  return response;
}
```

---

## XSS Prevention

### ❌ DON'T

```typescript
// Never trust user input for HTML rendering
<div dangerouslySetInnerHTML={{ __html: userInput }} />

// Never use eval with user data
eval(userProvidedCode);

// Never interpolate into scripts
<script>var data = {userInput};</script>
```

### ✅ DO

```typescript
// React auto-escapes by default
<div>{userInput}</div>

// If you MUST render HTML, sanitize first
import DOMPurify from 'dompurify';

const sanitizedHtml = DOMPurify.sanitize(userInput, {
  ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p'],
  ALLOWED_ATTR: ['href', 'target'],
});

<div dangerouslySetInnerHTML={{ __html: sanitizedHtml }} />

// Use textContent instead of innerHTML in vanilla JS
element.textContent = userInput;
```

---

## CSRF Protection

```typescript
// Next.js Server Actions have CSRF protection built-in
// For custom API routes:

// 1. SameSite cookies (default in modern browsers)
import { cookies } from 'next/headers';

cookies().set('session', token, {
  httpOnly: true,
  secure: process.env.NODE_ENV === 'production',
  sameSite: 'lax',  // or 'strict' for higher security
  maxAge: 60 * 60 * 24 * 7,  // 7 days
});

// 2. CSRF token pattern
import { randomBytes } from 'crypto';

export async function generateCsrfToken(): Promise<string> {
  return randomBytes(32).toString('hex');
}

// In form
<input type="hidden" name="csrf_token" value={csrfToken} />

// Verify on server
export async function POST(request: Request) {
  const formData = await request.formData();
  const csrfToken = formData.get('csrf_token');

  if (!await verifyCsrfToken(csrfToken)) {
    return Response.json({ error: 'Invalid CSRF token' }, { status: 403 });
  }
  // Process request...
}
```

---

## Input Validation

```typescript
// Always validate on SERVER (client validation is UX, not security)
import { z } from 'zod';

// Define strict schemas
const userSchema = z.object({
  email: z.string()
    .email('Invalid email format')
    .max(254, 'Email too long'),
  name: z.string()
    .min(1, 'Name required')
    .max(100, 'Name too long')
    .regex(/^[a-zA-Z\s\-']+$/, 'Invalid characters'),
  age: z.number()
    .int('Must be whole number')
    .positive('Must be positive')
    .max(150, 'Invalid age'),
  website: z.string()
    .url('Invalid URL')
    .optional(),
});

// Validate in API routes
export async function POST(request: Request) {
  const body = await request.json();
  const result = userSchema.safeParse(body);

  if (!result.success) {
    return Response.json({
      error: 'Validation failed',
      details: result.error.flatten(),
    }, { status: 400 });
  }

  // result.data is now typed and validated
  const { email, name, age } = result.data;
}
```

---

## SQL Injection Prevention

### ❌ DON'T

```typescript
// NEVER interpolate user input into queries
const query = `SELECT * FROM users WHERE email = '${email}'`;
const query = `SELECT * FROM users WHERE id = ${userId}`;
```

### ✅ DO

```typescript
// Prisma (parameterized by default)
const user = await prisma.user.findUnique({
  where: { email },
});

// Drizzle (parameterized)
const users = await db
  .select()
  .from(usersTable)
  .where(eq(usersTable.email, email));

// Raw SQL with parameterization
const user = await prisma.$queryRaw`
  SELECT * FROM users WHERE email = ${email}
`;

// PostgreSQL with pg (use placeholders)
const result = await client.query(
  'SELECT * FROM users WHERE email = $1',
  [email]
);
```

---

## Supply Chain Security (NEW for 2025)

```bash
# Lock dependencies
npm ci  # Use lockfile, don't update

# Audit dependencies
npm audit
npm audit fix

# Check for known vulnerabilities
npx better-npm-audit audit

# Use trusted registries
npm config set registry https://registry.npmjs.org/
```

```json
// package.json — Pin versions
{
  "dependencies": {
    "react": "19.0.0",        // Exact version
    "next": "~15.0.0",        // Patch updates only
    "lodash": "^4.17.21"      // Minor updates (be careful)
  },
  "overrides": {
    // Force secure version of transitive dependency
    "minimist": "^1.2.6"
  }
}
```

```typescript
// Verify package integrity
// package-lock.json includes integrity hashes

// Use Subresource Integrity for CDN scripts
<script
  src="https://cdn.example.com/lib.js"
  integrity="sha384-abc123..."
  crossorigin="anonymous"
/>
```

---

## Error Handling (NEW for 2025)

```typescript
// ❌ DON'T — Expose internal errors
app.get('/user/:id', async (req, res) => {
  try {
    const user = await db.getUser(req.params.id);
    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });  // Leaks info!
  }
});

// ✅ DO — Generic errors externally, detailed logs internally
app.get('/user/:id', async (req, res) => {
  try {
    const user = await db.getUser(req.params.id);
    res.json(user);
  } catch (error) {
    // Log detailed error internally
    logger.error('Database error', {
      error: error.message,
      stack: error.stack,
      userId: req.params.id,
      requestId: req.id,
    });

    // Return generic error to client
    res.status(500).json({
      error: 'An error occurred',
      requestId: req.id,  // For support reference
    });
  }
});
```

---

## Rate Limiting

```typescript
// Express with rate-limiter
import rateLimit from 'express-rate-limit';

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 100,                   // 100 requests per window
  standardHeaders: true,      // X-RateLimit-* headers
  legacyHeaders: false,
  message: {
    error: 'Too many requests',
    retryAfter: 15 * 60,
  },
});

// Stricter for auth endpoints
const authLimiter = rateLimit({
  windowMs: 60 * 1000,  // 1 minute
  max: 5,                // 5 attempts
});

app.use('/api/', limiter);
app.use('/api/auth/', authLimiter);
```

---

## Quick Reference

| Threat | Prevention |
|--------|------------|
| XSS | Auto-escape, CSP, DOMPurify |
| CSRF | SameSite cookies, tokens |
| SQL Injection | Parameterized queries, ORMs |
| Broken Access Control | RBAC, authorization checks |
| Security Misconfiguration | Security headers, audits |
| Supply Chain | Lock deps, audit, verify integrity |
| Cryptographic Failures | TLS 1.3, strong algorithms |
| Injection | Input validation, sanitization |
| Auth Failures | MFA, passkeys, rate limiting |
| Error Handling | Generic errors, internal logging |

| Header | Purpose |
|--------|---------|
| `Strict-Transport-Security` | Force HTTPS |
| `Content-Security-Policy` | Control resource loading |
| `X-Frame-Options` | Prevent clickjacking |
| `X-Content-Type-Options` | Prevent MIME sniffing |
| `Referrer-Policy` | Control referrer info |
| `Permissions-Policy` | Control browser features |

---

## Testing Security

```bash
# OWASP ZAP (automated scanning)
docker run -t owasp/zap2docker-stable zap-baseline.py -t https://your-site.com

# Security headers check
curl -I https://your-site.com | grep -i security

# SSL/TLS check
nmap --script ssl-enum-ciphers -p 443 your-site.com
```

---

## Resources

- [OWASP Top 10:2025](https://owasp.org/Top10/2025/)
- [OWASP API Security](https://owasp.org/www-project-api-security/)
- [MDN Security Headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers#security)
- [Content Security Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
- [GitLab OWASP 2025 Analysis](https://about.gitlab.com/blog/2025-owasp-top-10-whats-changed-and-why-it-matters/)
