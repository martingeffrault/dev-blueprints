# Secrets Management (2025)

> **Last updated**: January 2025
> **Scope**: Environment variables, vaults, .env handling

---

## Philosophy

Secrets should never be in code. Use environment variables, rotate regularly, and audit access.

---

## Environment Variables

### File Structure

```
.env                # Default values (committed, no secrets)
.env.local          # Local overrides (not committed)
.env.development    # Development environment
.env.production     # Production environment (not committed)
```

### .gitignore

```gitignore
.env*.local
.env.production
```

---

## Naming Convention

```bash
# Database
DATABASE_URL=postgresql://...

# API Keys (prefix with service)
STRIPE_SECRET_KEY=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...

# Auth
AUTH_SECRET=...
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...

# Third party
RESEND_API_KEY=...
UPSTASH_REDIS_URL=...
```

### Rules

- SCREAMING_SNAKE_CASE
- Prefix with service name
- Suffix with type (_KEY, _SECRET, _URL)

---

## Accessing in Code

```typescript
// Next.js
const apiKey = process.env.STRIPE_SECRET_KEY;

// With validation
import { z } from 'zod';

const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  STRIPE_SECRET_KEY: z.string().startsWith('sk_'),
});

export const env = envSchema.parse(process.env);
```

---

## Production Secrets

### ❌ DON'T

- Commit secrets to git
- Share via Slack/email
- Use same secrets for dev/prod
- Log secrets

### ✅ DO

- Use secret managers (Vercel, AWS, 1Password)
- Rotate regularly
- Audit access logs
- Use different secrets per environment

---

## Secret Managers

| Platform | Solution |
|----------|----------|
| Vercel | Environment Variables UI |
| AWS | Secrets Manager, Parameter Store |
| GCP | Secret Manager |
| Azure | Key Vault |
| Self-hosted | HashiCorp Vault, Doppler |

---

## Quick Reference

| File | Purpose | Committed |
|------|---------|-----------|
| `.env` | Defaults | Yes (no secrets) |
| `.env.local` | Local overrides | No |
| `.env.development` | Dev config | Maybe |
| `.env.production` | Prod secrets | No |
