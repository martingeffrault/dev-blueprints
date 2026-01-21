# Authentication (2025)

> **Last updated**: January 2025
> **Scope**: JWT, sessions, OAuth, best practices

---

## Philosophy

Authentication is the front door. Use established libraries — don't roll your own crypto. Prefer sessions over JWTs for web apps.

---

## Sessions vs JWTs

| Aspect | Sessions | JWTs |
|--------|----------|------|
| Storage | Server-side | Client-side |
| Revocation | Immediate | Requires blocklist |
| Size | Small cookie | Large token |
| Scalability | Requires shared store | Stateless |
| Best for | Web apps | APIs, mobile |

### Recommendation

- **Web apps**: Use sessions (Next-Auth, Lucia)
- **APIs**: Use JWTs with short expiry
- **Mobile**: Use JWTs with refresh tokens

---

## Session Best Practices

```typescript
// next-auth configuration
export const authOptions = {
  session: {
    strategy: 'database', // Not JWT
    maxAge: 30 * 24 * 60 * 60, // 30 days
  },
  cookies: {
    sessionToken: {
      options: {
        httpOnly: true,
        sameSite: 'lax',
        secure: process.env.NODE_ENV === 'production',
      },
    },
  },
};
```

---

## JWT Best Practices

```typescript
// Short-lived access tokens
const accessToken = jwt.sign(payload, secret, {
  expiresIn: '15m',
});

// Longer-lived refresh tokens (stored securely)
const refreshToken = jwt.sign({ userId }, refreshSecret, {
  expiresIn: '7d',
});
```

### ❌ DON'T

- Store JWTs in localStorage
- Use long expiry times
- Include sensitive data in payload

### ✅ DO

- Store in httpOnly cookies
- Use short expiry + refresh tokens
- Include only necessary claims

---

## OAuth 2.0

```typescript
// Use established libraries
import { Auth } from '@auth/core';
import Google from '@auth/core/providers/google';

export const auth = Auth({
  providers: [
    Google({
      clientId: process.env.GOOGLE_ID,
      clientSecret: process.env.GOOGLE_SECRET,
    }),
  ],
});
```

---

## Password Hashing

```typescript
// Use bcrypt or Argon2
import bcrypt from 'bcrypt';

// Hash
const hash = await bcrypt.hash(password, 12);

// Verify
const isValid = await bcrypt.compare(password, hash);
```

### ❌ NEVER

- Store plain text passwords
- Use MD5 or SHA1
- Roll your own hashing

---

## Quick Reference

| Pattern | Use |
|---------|-----|
| Sessions | Web apps with server |
| JWTs | APIs, microservices |
| OAuth | Third-party login |
| bcrypt | Password hashing |
| Argon2 | Password hashing (modern) |
| MFA | High-security apps |
