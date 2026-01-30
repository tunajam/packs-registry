# Next.js Context

Modern patterns for Next.js 14+ with App Router and Server Components.

## App Router Fundamentals

### File Conventions
```
app/
├── layout.tsx       # Root layout (required)
├── page.tsx         # Home page
├── loading.tsx      # Loading UI
├── error.tsx        # Error handling
├── not-found.tsx    # 404 page
├── route.ts         # API route handler
└── [slug]/          # Dynamic segment
    └── page.tsx
```

### Server vs Client Components

**Server Components (default)**
- Can fetch data directly
- Access backend resources
- Keep sensitive data secure
- No JavaScript sent to client

```tsx
// app/users/page.tsx (Server Component by default)
async function UsersPage() {
  const users = await db.users.findMany()
  return <UserList users={users} />
}
```

**Client Components**
- Use `"use client"` directive
- Access browser APIs
- Use hooks (useState, useEffect)
- Handle user interactions

```tsx
"use client"

import { useState } from "react"

export function Counter() {
  const [count, setCount] = useState(0)
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>
}
```

### Data Fetching

**Server Components with async/await:**
```tsx
async function Page({ params }: { params: { id: string } }) {
  const product = await getProduct(params.id)
  return <ProductDetails product={product} />
}
```

**Parallel Data Fetching:**
```tsx
async function Page() {
  const [user, posts] = await Promise.all([
    getUser(),
    getPosts()
  ])
  return <Dashboard user={user} posts={posts} />
}
```

### Server Actions

```tsx
// app/actions.ts
"use server"

import { revalidatePath } from "next/cache"

export async function createPost(formData: FormData) {
  const title = formData.get("title")
  await db.posts.create({ data: { title } })
  revalidatePath("/posts")
}
```

```tsx
// app/posts/new/page.tsx
import { createPost } from "./actions"

export default function NewPost() {
  return (
    <form action={createPost}>
      <input name="title" />
      <button type="submit">Create</button>
    </form>
  )
}
```

### Metadata

```tsx
import type { Metadata } from "next"

export const metadata: Metadata = {
  title: "My App",
  description: "Description here",
  openGraph: {
    title: "My App",
    images: ["/og.png"],
  },
}
```

**Dynamic Metadata:**
```tsx
export async function generateMetadata({ params }): Promise<Metadata> {
  const product = await getProduct(params.id)
  return {
    title: product.name,
    description: product.description,
  }
}
```

### Route Handlers (API)

```tsx
// app/api/users/route.ts
import { NextResponse } from "next/server"

export async function GET() {
  const users = await db.users.findMany()
  return NextResponse.json(users)
}

export async function POST(request: Request) {
  const data = await request.json()
  const user = await db.users.create({ data })
  return NextResponse.json(user, { status: 201 })
}
```

### Caching & Revalidation

```tsx
// Time-based revalidation
fetch(url, { next: { revalidate: 3600 } }) // 1 hour

// On-demand revalidation
import { revalidatePath, revalidateTag } from "next/cache"

revalidatePath("/posts")          // Revalidate path
revalidateTag("posts")            // Revalidate tagged fetches
```

### Middleware

```tsx
// middleware.ts (root level)
import { NextResponse } from "next/server"
import type { NextRequest } from "next/server"

export function middleware(request: NextRequest) {
  if (!request.cookies.get("token")) {
    return NextResponse.redirect(new URL("/login", request.url))
  }
}

export const config = {
  matcher: ["/dashboard/:path*", "/api/protected/:path*"],
}
```

## Common Patterns

### Loading States
```tsx
// app/products/loading.tsx
export default function Loading() {
  return <ProductSkeleton />
}
```

### Error Handling
```tsx
// app/products/error.tsx
"use client"

export default function Error({
  error,
  reset,
}: {
  error: Error
  reset: () => void
}) {
  return (
    <div>
      <h2>Something went wrong!</h2>
      <button onClick={() => reset()}>Try again</button>
    </div>
  )
}
```

### Streaming with Suspense
```tsx
import { Suspense } from "react"

export default function Page() {
  return (
    <>
      <Header />
      <Suspense fallback={<PostsSkeleton />}>
        <Posts />
      </Suspense>
    </>
  )
}
```

## Best Practices

1. **Keep Server Components as default** — only use "use client" when needed
2. **Fetch data in Server Components** — avoid useEffect for data fetching
3. **Use Server Actions for mutations** — better than API routes for forms
4. **Colocate loading/error UI** — use loading.tsx and error.tsx files
5. **Use generateStaticParams** — for static generation of dynamic routes
6. **Implement proper caching** — use revalidate options appropriately
