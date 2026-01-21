# Authentication (2025)

> **Last updated**: January 2026
> **Scope**: Passkeys, JWT, sessions, OAuth 2.0, MFA
> **Purpose**: Modern authentication patterns and best practices

---

## Philosophy

Authentication is the front door. In 2025, passkeys are mainstream — 3 billion in active use, 69% of users have at least one. Use established libraries, never roll your own crypto. Prioritize phishing-resistant methods.

---

## Authentication Methods (2025 Priority)

| Method | Security | UX | Use Case |
|--------|----------|-----|----------|
| **Passkeys** | Excellent | Excellent | Primary auth (recommended) |
| **Hardware Keys** | Excellent | Fair | High-security, regulated |
| **OAuth 2.0** | Good | Excellent | Social login, SSO |
| **Sessions** | Good | Good | Web apps with server |
| **JWTs** | Good | Good | APIs, mobile apps |
| **Passwords + MFA** | Fair | Fair | Legacy systems |

---

## Passkeys (WebAuthn/FIDO2)

Passkeys are phishing-resistant, cryptographic credentials that replace passwords. NIST SP 800-63-4 (July 2025) mandates phishing-resistant options for AAL2.

### Implementation

```typescript
// Using SimpleWebAuthn (recommended library)
import {
  generateRegistrationOptions,
  verifyRegistrationResponse,
  generateAuthenticationOptions,
  verifyAuthenticationResponse,
} from '@simplewebauthn/server';

// Registration - Generate options
export async function POST(request: Request) {
  const { userId, username, email } = await request.json();

  const options = await generateRegistrationOptions({
    rpName: 'My App',
    rpID: 'example.com',
    userID: userId,
    userName: email,
    userDisplayName: username,
    attestationType: 'none',  // 'direct' for enterprise
    authenticatorSelection: {
      residentKey: 'preferred',  // Discoverable credential
      userVerification: 'preferred',
      authenticatorAttachment: 'platform',  // Built-in biometrics
    },
  });

  // Store challenge in session/DB
  await storeChallenge(userId, options.challenge);

  return Response.json(options);
}

// Registration - Verify response
export async function PUT(request: Request) {
  const { userId, credential } = await request.json();
  const expectedChallenge = await getStoredChallenge(userId);

  const verification = await verifyRegistrationResponse({
    response: credential,
    expectedChallenge,
    expectedOrigin: 'https://example.com',
    expectedRPID: 'example.com',
  });

  if (verification.verified) {
    // Store credential for user
    await saveCredential(userId, {
      credentialID: verification.registrationInfo.credentialID,
      credentialPublicKey: verification.registrationInfo.credentialPublicKey,
      counter: verification.registrationInfo.counter,
    });
  }

  return Response.json({ verified: verification.verified });
}
```

### Client-Side (React)

```typescript
import { startRegistration, startAuthentication } from '@simplewebauthn/browser';

// Register passkey
async function registerPasskey() {
  // Get options from server
  const optionsRes = await fetch('/api/auth/passkey/register', {
    method: 'POST',
    body: JSON.stringify({ userId, username, email }),
  });
  const options = await optionsRes.json();

  // Prompt user for biometric/PIN
  const credential = await startRegistration(options);

  // Send to server for verification
  const verifyRes = await fetch('/api/auth/passkey/register', {
    method: 'PUT',
    body: JSON.stringify({ userId, credential }),
  });

  return verifyRes.json();
}

// Authenticate with passkey
async function authenticateWithPasskey() {
  const optionsRes = await fetch('/api/auth/passkey/authenticate', {
    method: 'POST',
  });
  const options = await optionsRes.json();

  const credential = await startAuthentication(options);

  const verifyRes = await fetch('/api/auth/passkey/authenticate', {
    method: 'PUT',
    body: JSON.stringify({ credential }),
  });

  return verifyRes.json();
}
```

### Passkey Best Practices

```typescript
// ✅ DO
// 1. Offer passkey registration after successful login
// 2. Support multiple passkeys per user (backup)
// 3. Provide fallback to other auth methods
// 4. Use conditional UI for discoverable credentials

// Conditional UI (autofill-like experience)
const credential = await startAuthentication(options, {
  conditionalUI: true,
});

// ❌ DON'T
// - Force passkey-only without fallback
// - Delete user's only passkey without confirmation
// - Skip user verification for sensitive actions
```

---

## Sessions vs JWTs

| Aspect | Sessions | JWTs |
|--------|----------|------|
| Storage | Server-side | Client-side |
| Revocation | Immediate | Requires blocklist |
| Size | Small cookie | Large token |
| Scalability | Requires shared store | Stateless |
| Best for | Web apps | APIs, mobile |

### Session Best Practices (2025)

```typescript
// Auth.js (formerly NextAuth.js) v5
import NextAuth from 'next-auth';
import { DrizzleAdapter } from '@auth/drizzle-adapter';
import Google from 'next-auth/providers/google';
import Passkey from 'next-auth/providers/passkey';

export const { handlers, auth, signIn, signOut } = NextAuth({
  adapter: DrizzleAdapter(db),
  providers: [
    Passkey,  // Passkey support built-in
    Google,
  ],
  session: {
    strategy: 'database',  // Server-side sessions
    maxAge: 30 * 24 * 60 * 60,  // 30 days
  },
  cookies: {
    sessionToken: {
      name: '__Secure-authjs.session-token',
      options: {
        httpOnly: true,
        sameSite: 'lax',
        path: '/',
        secure: true,
      },
    },
  },
  experimental: {
    enableWebAuthn: true,  // Enable passkeys
  },
});

// Protect routes
import { auth } from '@/auth';

export default async function ProtectedPage() {
  const session = await auth();

  if (!session) {
    redirect('/login');
  }

  return <div>Welcome, {session.user.name}</div>;
}
```

### JWT Best Practices (2025)

```typescript
import { SignJWT, jwtVerify } from 'jose';

// Create keys (use RS256 or ES256 for production)
const secret = new TextEncoder().encode(process.env.JWT_SECRET);

// Generate access token (short-lived)
async function generateAccessToken(userId: string, roles: string[]) {
  return new SignJWT({
    sub: userId,
    roles,
    type: 'access',
  })
    .setProtectedHeader({ alg: 'HS256' })
    .setIssuedAt()
    .setExpirationTime('15m')  // Short lived!
    .setIssuer('https://example.com')
    .setAudience('https://api.example.com')
    .sign(secret);
}

// Generate refresh token (longer-lived, stored securely)
async function generateRefreshToken(userId: string) {
  const token = new SignJWT({
    sub: userId,
    type: 'refresh',
  })
    .setProtectedHeader({ alg: 'HS256' })
    .setIssuedAt()
    .setExpirationTime('7d')
    .setJti(crypto.randomUUID())  // Unique ID for revocation
    .sign(secret);

  // Store in DB for revocation capability
  await db.refreshToken.create({
    data: {
      userId,
      tokenId: jti,
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
    },
  });

  return token;
}

// Verify token
async function verifyToken(token: string) {
  try {
    const { payload } = await jwtVerify(token, secret, {
      issuer: 'https://example.com',
      audience: 'https://api.example.com',
    });
    return payload;
  } catch {
    return null;
  }
}

// Token refresh endpoint
export async function POST(request: Request) {
  const { refreshToken } = await request.json();

  const payload = await verifyToken(refreshToken);
  if (!payload || payload.type !== 'refresh') {
    return Response.json({ error: 'Invalid token' }, { status: 401 });
  }

  // Check if token is revoked
  const storedToken = await db.refreshToken.findUnique({
    where: { tokenId: payload.jti },
  });

  if (!storedToken || storedToken.revoked) {
    return Response.json({ error: 'Token revoked' }, { status: 401 });
  }

  // Generate new tokens
  const accessToken = await generateAccessToken(payload.sub, payload.roles);

  return Response.json({ accessToken });
}
```

### JWT Anti-Patterns

```typescript
// ❌ DON'T
localStorage.setItem('token', jwt);  // XSS vulnerable!
const token = jwt.sign(user, secret);  // No expiration!
jwt.sign({ password: user.password }, secret);  // Sensitive data!

// ✅ DO
// Store in httpOnly cookie
// Always set expiration
// Only include necessary claims (sub, roles)
```

---

## OAuth 2.0 / OIDC

```typescript
// Auth.js with multiple providers
import GitHub from 'next-auth/providers/github';
import Google from 'next-auth/providers/google';
import Credentials from 'next-auth/providers/credentials';

export const { handlers, auth } = NextAuth({
  providers: [
    GitHub({
      clientId: process.env.GITHUB_ID,
      clientSecret: process.env.GITHUB_SECRET,
    }),
    Google({
      clientId: process.env.GOOGLE_ID,
      clientSecret: process.env.GOOGLE_SECRET,
      authorization: {
        params: {
          prompt: 'consent',
          access_type: 'offline',
          response_type: 'code',
        },
      },
    }),
  ],
  callbacks: {
    async signIn({ user, account, profile }) {
      // Custom logic (e.g., check allowed domains)
      if (account?.provider === 'google') {
        return profile?.email?.endsWith('@company.com') ?? false;
      }
      return true;
    },
    async session({ session, user }) {
      // Add user ID to session
      session.user.id = user.id;
      return session;
    },
  },
});
```

---

## Password Hashing

```typescript
// Use Argon2 (recommended) or bcrypt
import { hash, verify } from '@node-rs/argon2';

// Hash password
async function hashPassword(password: string): Promise<string> {
  return hash(password, {
    memoryCost: 65536,    // 64 MiB
    timeCost: 3,          // 3 iterations
    parallelism: 4,       // 4 parallel threads
    outputLen: 32,        // 32 bytes output
  });
}

// Verify password
async function verifyPassword(password: string, hash: string): Promise<boolean> {
  try {
    return await verify(hash, password);
  } catch {
    return false;
  }
}

// Using bcrypt (alternative)
import bcrypt from 'bcrypt';

const SALT_ROUNDS = 12;  // 2025 recommendation

async function hashPasswordBcrypt(password: string): Promise<string> {
  return bcrypt.hash(password, SALT_ROUNDS);
}

async function verifyPasswordBcrypt(password: string, hash: string): Promise<boolean> {
  return bcrypt.compare(password, hash);
}
```

### ❌ NEVER

- Store plain text passwords
- Use MD5, SHA1, SHA256 alone (no salt, too fast)
- Use fewer than 10 bcrypt rounds
- Roll your own hashing algorithm

---

## Multi-Factor Authentication (MFA)

```typescript
// TOTP (Time-based One-Time Password)
import { authenticator } from 'otplib';

// Generate secret for user
function generateTotpSecret(): string {
  return authenticator.generateSecret();
}

// Generate QR code URL
function generateTotpUri(secret: string, email: string): string {
  return authenticator.keyuri(email, 'My App', secret);
}

// Verify TOTP code
function verifyTotp(token: string, secret: string): boolean {
  return authenticator.verify({ token, secret });
}

// Setup flow
export async function POST(request: Request) {
  const { userId } = await auth();
  const secret = generateTotpSecret();

  // Store encrypted secret temporarily
  await db.user.update({
    where: { id: userId },
    data: { totpSecretTemp: encrypt(secret) },
  });

  const uri = generateTotpUri(secret, user.email);
  const qrCode = await QRCode.toDataURL(uri);

  return Response.json({ qrCode, secret });
}

// Verify and enable
export async function PUT(request: Request) {
  const { code } = await request.json();
  const { userId } = await auth();

  const user = await db.user.findUnique({ where: { id: userId } });
  const secret = decrypt(user.totpSecretTemp);

  if (!verifyTotp(code, secret)) {
    return Response.json({ error: 'Invalid code' }, { status: 400 });
  }

  // Enable MFA
  await db.user.update({
    where: { id: userId },
    data: {
      totpSecret: user.totpSecretTemp,
      totpSecretTemp: null,
      mfaEnabled: true,
    },
  });

  // Generate backup codes
  const backupCodes = generateBackupCodes();
  await storeBackupCodes(userId, backupCodes);

  return Response.json({ backupCodes });
}
```

---

## Rate Limiting for Auth

```typescript
// Strict rate limiting for authentication endpoints
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(5, '1 m'),  // 5 attempts per minute
  analytics: true,
});

export async function POST(request: Request) {
  const ip = request.headers.get('x-forwarded-for') ?? 'unknown';
  const { success, remaining, reset } = await ratelimit.limit(ip);

  if (!success) {
    return Response.json(
      { error: 'Too many attempts', retryAfter: reset },
      {
        status: 429,
        headers: {
          'Retry-After': String(reset),
          'X-RateLimit-Remaining': String(remaining),
        },
      }
    );
  }

  // Process login...
}
```

---

## Quick Reference

| Pattern | Use Case | Library |
|---------|----------|---------|
| Passkeys | Primary auth | SimpleWebAuthn |
| Sessions | Web apps | Auth.js |
| JWTs | APIs, mobile | jose |
| OAuth | Social login | Auth.js |
| TOTP | MFA | otplib |
| Password hash | Legacy auth | Argon2, bcrypt |

| Security Measure | Implementation |
|------------------|----------------|
| Short token expiry | Access: 15m, Refresh: 7d |
| HttpOnly cookies | `httpOnly: true` |
| SameSite cookies | `sameSite: 'lax'` |
| Secure cookies | `secure: true` (production) |
| Rate limiting | 5 attempts/minute for auth |
| Brute force protection | Account lockout after failures |

---

## Resources

- [Passkeys Developer Guide](https://passkeys.dev/)
- [SimpleWebAuthn Documentation](https://simplewebauthn.dev/)
- [Auth.js Documentation](https://authjs.dev/)
- [NIST SP 800-63-4](https://pages.nist.gov/800-63-4/)
- [Passkeys State 2025](https://www.1password.community/blog/random-but-memorable/the-state-of-passkeys-in-2025/163464)
- [WebAuthn.io Testing Tool](https://webauthn.io/)
- [FIDO Alliance](https://fidoalliance.org/passkeys/)
