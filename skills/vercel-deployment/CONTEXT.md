# Vercel Deployment Context

Comprehensive guide to Vercel deployment, configuration, and best practices.

## vercel.json Configuration

```json
{
  "$schema": "https://openapi.vercel.sh/vercel.json",
  "framework": "nextjs",
  "buildCommand": "npm run build",
  "outputDirectory": ".next",
  "installCommand": "npm ci",
  "devCommand": "npm run dev",
  
  "regions": ["iad1", "sfo1"],
  
  "headers": [
    {
      "source": "/api/(.*)",
      "headers": [
        { "key": "Access-Control-Allow-Origin", "value": "*" },
        { "key": "Cache-Control", "value": "no-store" }
      ]
    },
    {
      "source": "/(.*)",
      "headers": [
        { "key": "X-Frame-Options", "value": "DENY" },
        { "key": "X-Content-Type-Options", "value": "nosniff" }
      ]
    }
  ],
  
  "redirects": [
    {
      "source": "/old-path",
      "destination": "/new-path",
      "permanent": true
    },
    {
      "source": "/blog/:slug",
      "destination": "/posts/:slug",
      "permanent": false
    }
  ],
  
  "rewrites": [
    {
      "source": "/api/:path*",
      "destination": "https://api.example.com/:path*"
    },
    {
      "source": "/:path*",
      "has": [{ "type": "host", "value": "app.example.com" }],
      "destination": "/app/:path*"
    }
  ],
  
  "functions": {
    "api/**/*.ts": {
      "maxDuration": 30
    },
    "api/heavy-task.ts": {
      "memory": 1024,
      "maxDuration": 60
    }
  },
  
  "crons": [
    {
      "path": "/api/cron/daily",
      "schedule": "0 0 * * *"
    },
    {
      "path": "/api/cron/hourly",
      "schedule": "0 * * * *"
    }
  ]
}
```

## Environment Variables

### CLI Management
```bash
# Add environment variable
vercel env add DATABASE_URL production

# Add to multiple environments
vercel env add API_KEY production preview development

# Pull env vars to local .env
vercel env pull .env.local

# List all env vars
vercel env ls

# Remove
vercel env rm SECRET_KEY production
```

### In vercel.json
```json
{
  "env": {
    "NEXT_PUBLIC_API_URL": "https://api.example.com"
  },
  "build": {
    "env": {
      "CI": "true"
    }
  }
}
```

### Environment-Specific
```
Production:  NEXT_PUBLIC_API_URL=https://api.example.com
Preview:     NEXT_PUBLIC_API_URL=https://staging-api.example.com
Development: NEXT_PUBLIC_API_URL=http://localhost:8000
```

## Edge Functions

### Basic Edge Function
```typescript
// app/api/edge/route.ts
import { NextRequest } from 'next/server'

export const runtime = 'edge'

export async function GET(request: NextRequest) {
  const country = request.geo?.country || 'Unknown'
  const city = request.geo?.city || 'Unknown'
  
  return Response.json({
    message: `Hello from ${city}, ${country}!`,
    timestamp: Date.now(),
  })
}
```

### Edge Middleware
```typescript
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export const config = {
  matcher: [
    '/((?!_next/static|_next/image|favicon.ico).*)',
  ],
}

export function middleware(request: NextRequest) {
  const response = NextResponse.next()
  
  // Add custom headers
  response.headers.set('x-custom-header', 'value')
  
  // Geolocation-based routing
  const country = request.geo?.country
  if (country === 'DE') {
    return NextResponse.redirect(new URL('/de', request.url))
  }
  
  // A/B testing
  const bucket = request.cookies.get('bucket')?.value || 
    (Math.random() > 0.5 ? 'a' : 'b')
  
  if (!request.cookies.has('bucket')) {
    response.cookies.set('bucket', bucket)
  }
  
  return response
}
```

### Edge Config
```typescript
import { get } from '@vercel/edge-config'

export const runtime = 'edge'

export async function GET() {
  const greeting = await get('greeting')
  const featureFlags = await get('featureFlags')
  
  return Response.json({ greeting, featureFlags })
}
```

## Serverless Functions

### API Route with Streaming
```typescript
// app/api/stream/route.ts
export async function GET() {
  const encoder = new TextEncoder()
  
  const stream = new ReadableStream({
    async start(controller) {
      for (let i = 0; i < 10; i++) {
        controller.enqueue(encoder.encode(`data: ${i}\n\n`))
        await new Promise(r => setTimeout(r, 100))
      }
      controller.close()
    },
  })
  
  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
    },
  })
}
```

### Function Configuration
```typescript
// app/api/heavy/route.ts

// Runtime configuration
export const runtime = 'nodejs' // or 'edge'
export const maxDuration = 60 // seconds (Pro/Enterprise)
export const dynamic = 'force-dynamic'

export async function POST(request: Request) {
  // Long-running task
  const result = await heavyComputation()
  return Response.json(result)
}
```

## Caching & Performance

### Cache Headers
```typescript
export async function GET() {
  const data = await fetchData()
  
  return Response.json(data, {
    headers: {
      // Cache for 1 hour, stale-while-revalidate for 1 day
      'Cache-Control': 'public, s-maxage=3600, stale-while-revalidate=86400',
    },
  })
}
```

### ISR (Incremental Static Regeneration)
```typescript
// app/posts/[slug]/page.tsx
export const revalidate = 3600 // Revalidate every hour

export async function generateStaticParams() {
  const posts = await getPosts()
  return posts.map((post) => ({ slug: post.slug }))
}
```

### On-Demand Revalidation
```typescript
// app/api/revalidate/route.ts
import { revalidatePath, revalidateTag } from 'next/cache'

export async function POST(request: Request) {
  const { path, tag, secret } = await request.json()
  
  if (secret !== process.env.REVALIDATION_SECRET) {
    return Response.json({ error: 'Invalid secret' }, { status: 401 })
  }
  
  if (path) {
    revalidatePath(path)
  }
  if (tag) {
    revalidateTag(tag)
  }
  
  return Response.json({ revalidated: true, now: Date.now() })
}
```

## Preview Deployments

### Branch-based URLs
```
main branch:     myapp.vercel.app
feature/xyz:     myapp-feature-xyz-team.vercel.app
PR #123:         myapp-git-pr-123-team.vercel.app
```

### Preview Protection
```json
{
  "git": {
    "deploymentEnabled": {
      "main": true,
      "feature/*": true
    }
  }
}
```

## Monorepo Configuration

```json
{
  "framework": "nextjs",
  "installCommand": "pnpm install",
  "buildCommand": "pnpm turbo build --filter=web",
  "outputDirectory": "apps/web/.next",
  "ignoreCommand": "npx turbo-ignore"
}
```

### Root Configuration (vercel.json)
```json
{
  "projects": [
    {
      "name": "web",
      "framework": "nextjs",
      "rootDirectory": "apps/web"
    },
    {
      "name": "api",
      "framework": null,
      "rootDirectory": "apps/api"
    }
  ]
}
```

## Security Headers

```json
{
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        {
          "key": "Content-Security-Policy",
          "value": "default-src 'self'; script-src 'self' 'unsafe-eval' 'unsafe-inline'; style-src 'self' 'unsafe-inline';"
        },
        {
          "key": "X-Frame-Options",
          "value": "SAMEORIGIN"
        },
        {
          "key": "X-Content-Type-Options",
          "value": "nosniff"
        },
        {
          "key": "Referrer-Policy",
          "value": "strict-origin-when-cross-origin"
        },
        {
          "key": "Permissions-Policy",
          "value": "camera=(), microphone=(), geolocation=()"
        }
      ]
    }
  ]
}
```

## CLI Commands

```bash
# Deploy
vercel                    # Deploy to preview
vercel --prod             # Deploy to production
vercel deploy --prebuilt  # Deploy pre-built output

# Development
vercel dev                # Run development server with Vercel features
vercel build              # Build locally

# Project management
vercel link               # Link to existing project
vercel project ls         # List projects
vercel domains            # Manage domains

# Logs & debugging
vercel logs               # View deployment logs
vercel inspect            # Inspect deployment
```

## Best Practices

1. **Use Edge for low-latency** - Geolocation, A/B tests, auth checks
2. **Cache aggressively** - Use ISR and cache headers
3. **Minimize cold starts** - Keep functions small
4. **Environment separation** - Different env vars per environment
5. **Use preview deployments** - Test PRs before merging
6. **Monitor function duration** - Stay under limits
7. **Leverage Edge Config** - For feature flags and dynamic config
