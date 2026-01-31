# Environment Variable Validation Context

Comprehensive guide to environment variable management, validation, and security.

## The Problem

Environment variables are stringly-typed, easy to misspell, and fail silently at runtime. Type-safe validation catches issues at build time.

## t3-env (Recommended)

Type-safe environment variables with Zod validation.

### Installation

```bash
npm install @t3-oss/env-nextjs zod
# Or for core (framework-agnostic)
npm install @t3-oss/env-core zod
```

### Basic Setup (Next.js)

```typescript
// src/env.ts
import { createEnv } from "@t3-oss/env-nextjs"
import { z } from "zod"

export const env = createEnv({
  /**
   * Server-side environment variables schema
   */
  server: {
    DATABASE_URL: z.string().url(),
    NODE_ENV: z.enum(["development", "test", "production"]).default("development"),
    OPENAI_API_KEY: z.string().min(1),
    STRIPE_SECRET_KEY: z.string().startsWith("sk_"),
    REDIS_URL: z.string().url().optional(),
  },

  /**
   * Client-side environment variables schema
   * Must be prefixed with NEXT_PUBLIC_
   */
  client: {
    NEXT_PUBLIC_APP_URL: z.string().url(),
    NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY: z.string().startsWith("pk_"),
    NEXT_PUBLIC_POSTHOG_KEY: z.string().optional(),
  },

  /**
   * Destructure process.env here
   * Can't destructure at runtime due to Next.js bundling
   */
  runtimeEnv: {
    DATABASE_URL: process.env.DATABASE_URL,
    NODE_ENV: process.env.NODE_ENV,
    OPENAI_API_KEY: process.env.OPENAI_API_KEY,
    STRIPE_SECRET_KEY: process.env.STRIPE_SECRET_KEY,
    REDIS_URL: process.env.REDIS_URL,
    NEXT_PUBLIC_APP_URL: process.env.NEXT_PUBLIC_APP_URL,
    NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY: process.env.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY,
    NEXT_PUBLIC_POSTHOG_KEY: process.env.NEXT_PUBLIC_POSTHOG_KEY,
  },

  /**
   * Skip validation in certain environments
   */
  skipValidation: !!process.env.SKIP_ENV_VALIDATION,

  /**
   * Empty strings are treated as undefined
   */
  emptyStringAsUndefined: true,
})
```

### Usage

```typescript
// Server-side
import { env } from "@/env"

// Type-safe access
const db = new Database(env.DATABASE_URL)
const stripe = new Stripe(env.STRIPE_SECRET_KEY)

// Client-side (only NEXT_PUBLIC_ vars available)
console.log(env.NEXT_PUBLIC_APP_URL)
```

### Shared Variables

```typescript
export const env = createEnv({
  shared: {
    NODE_ENV: z.enum(["development", "test", "production"]),
    VERCEL_URL: z.string().optional(),
  },
  server: { /* ... */ },
  client: { /* ... */ },
  runtimeEnv: { /* ... */ },
})
```

## Pure Zod Approach

For non-Next.js projects or simpler setups:

```typescript
// src/env.ts
import { z } from "zod"

const envSchema = z.object({
  NODE_ENV: z.enum(["development", "test", "production"]).default("development"),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  API_KEY: z.string().min(1),
  ENABLE_FEATURE: z.string().transform((s) => s === "true").default("false"),
  ALLOWED_ORIGINS: z.string().transform((s) => s.split(",")).default(""),
})

export type Env = z.infer<typeof envSchema>

function validateEnv(): Env {
  const parsed = envSchema.safeParse(process.env)
  
  if (!parsed.success) {
    console.error("❌ Invalid environment variables:")
    console.error(parsed.error.flatten().fieldErrors)
    throw new Error("Invalid environment variables")
  }
  
  return parsed.data
}

export const env = validateEnv()
```

## .env File Patterns

### File Structure

```
.env                 # Default, committed (safe defaults only)
.env.local           # Local overrides, gitignored
.env.development     # Dev environment
.env.production      # Production (don't commit secrets!)
.env.test            # Test environment
.env.example         # Template with all required vars
```

### .env.example

```bash
# .env.example - Commit this file!
# Copy to .env.local and fill in values

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/mydb

# Authentication
AUTH_SECRET=generate-with-openssl-rand-base64-32
NEXTAUTH_URL=http://localhost:3000

# External Services
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
OPENAI_API_KEY=sk-...

# Feature Flags
ENABLE_NEW_DASHBOARD=false

# Client-side (public)
NEXT_PUBLIC_APP_URL=http://localhost:3000
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_...
```

### .gitignore

```gitignore
# Environment files
.env
.env.local
.env.*.local
.env.development.local
.env.test.local
.env.production.local

# Allow example file
!.env.example
```

## Common Zod Patterns for Env

```typescript
const envSchema = z.object({
  // Required string
  API_KEY: z.string().min(1),
  
  // URL validation
  DATABASE_URL: z.string().url(),
  
  // Enum
  NODE_ENV: z.enum(["development", "test", "production"]),
  
  // Number coercion
  PORT: z.coerce.number().int().positive().default(3000),
  
  // Boolean from string
  DEBUG: z.string().transform((s) => s === "true").default("false"),
  
  // Optional with default
  LOG_LEVEL: z.enum(["debug", "info", "warn", "error"]).default("info"),
  
  // Comma-separated array
  ALLOWED_HOSTS: z.string().transform((s) => s.split(",").filter(Boolean)),
  
  // JSON parsing
  FEATURE_FLAGS: z.string().transform((s) => JSON.parse(s)),
  
  // Prefix validation
  STRIPE_KEY: z.string().startsWith("sk_"),
  
  // Length constraints
  SECRET_KEY: z.string().min(32, "Secret key must be at least 32 characters"),
  
  // Regex validation
  AWS_REGION: z.string().regex(/^[a-z]{2}-[a-z]+-\d$/),
  
  // Union types
  CACHE_DRIVER: z.union([z.literal("redis"), z.literal("memory")]),
  
  // Nullable/Optional
  SENTRY_DSN: z.string().url().optional(),
})
```

## Security Best Practices

### 1. Never Commit Secrets
```bash
# Use secret managers instead
# AWS Secrets Manager, Vercel, 1Password CLI, etc.

# Check for secrets in git history
git log -p | grep -i "api_key\|secret\|password"
```

### 2. Validate Early
```typescript
// Validate at app startup, not when first accessed
// src/index.ts
import "./env" // Import to trigger validation
```

### 3. Use Environment-Specific Files
```typescript
// Load correct env file based on NODE_ENV
import { config } from "dotenv"

config({
  path: `.env.${process.env.NODE_ENV || "development"}.local`,
})
```

### 4. Mask Sensitive Values in Logs
```typescript
function maskSecrets(env: Record<string, string>) {
  const sensitiveKeys = ["KEY", "SECRET", "TOKEN", "PASSWORD"]
  return Object.fromEntries(
    Object.entries(env).map(([key, value]) => [
      key,
      sensitiveKeys.some((s) => key.includes(s)) ? "***" : value,
    ])
  )
}
```

### 5. Runtime vs Build Time
```typescript
// Build-time: Inlined into bundle (DANGEROUS for secrets!)
const apiKey = process.env.NEXT_PUBLIC_API_KEY // Client-visible!

// Runtime: Read at request time (safe for secrets)
const secret = process.env.SECRET_KEY // Server-only
```

## Deployment Patterns

### Vercel
```bash
# CLI
vercel env add DATABASE_URL production
vercel env pull .env.local

# vercel.json
{
  "env": {
    "NEXT_PUBLIC_APP_URL": "https://myapp.vercel.app"
  }
}
```

### Docker
```dockerfile
# Don't bake secrets into images!
FROM node:20-alpine
# Use runtime env injection
ENV NODE_ENV=production
CMD ["node", "dist/index.js"]

# docker-compose.yml
services:
  app:
    env_file: .env.production
    environment:
      - DATABASE_URL=${DATABASE_URL}
```

### GitHub Actions
```yaml
jobs:
  deploy:
    env:
      DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

## Quick Checklist

- [ ] All env vars validated at startup
- [ ] .env.example committed with all required vars
- [ ] Sensitive files in .gitignore
- [ ] Client vs server vars clearly separated
- [ ] Type-safe access throughout codebase
- [ ] Secrets stored in secret manager, not files
- [ ] CI/CD has all required env vars configured
