# Next.js App Router Context

Comprehensive guide to Next.js App Router, Server Components, and modern patterns.

## Project Structure

```
app/
├── layout.tsx           # Root layout (required)
├── page.tsx             # Home page (/)
├── loading.tsx          # Loading UI
├── error.tsx            # Error UI
├── not-found.tsx        # 404 page
├── global-error.tsx     # Global error boundary
├── (auth)/              # Route group (no URL impact)
│   ├── login/page.tsx   # /login
│   └── register/page.tsx # /register
├── dashboard/
│   ├── layout.tsx       # Dashboard layout
│   ├── page.tsx         # /dashboard
│   └── settings/page.tsx # /dashboard/settings
├── blog/
│   ├── page.tsx         # /blog
│   └── [slug]/
│       ├── page.tsx     # /blog/[slug]
│       └── opengraph-image.tsx
├── api/
│   └── route.ts         # API route
└── [...catchAll]/
    └── page.tsx         # Catch-all route
```

## Server Components (Default)

```tsx
// app/users/page.tsx - Server Component by default
import { db } from '@/lib/db'

export default async function UsersPage() {
  // Direct database access - no API needed!
  const users = await db.user.findMany()
  
  return (
    <div>
      <h1>Users</h1>
      <ul>
        {users.map(user => (
          <li key={user.id}>{user.name}</li>
        ))}
      </ul>
    </div>
  )
}
```

## Client Components

```tsx
'use client' // Must be at the top

import { useState } from 'react'

export function Counter() {
  const [count, setCount] = useState(0)
  
  return (
    <button onClick={() => setCount(c => c + 1)}>
      Count: {count}
    </button>
  )
}
```

## When to Use Client vs Server

| Server Components | Client Components |
|-------------------|-------------------|
| Fetch data | useState, useEffect |
| Access backend directly | Event listeners (onClick) |
| Keep sensitive logic server-side | Browser APIs |
| Large dependencies (render on server) | Interactive UI |
| SEO-critical content | Third-party client libs |

## Layouts

### Root Layout (Required)
```tsx
// app/layout.tsx
import { Inter } from 'next/font/google'
import './globals.css'

const inter = Inter({ subsets: ['latin'] })

export const metadata = {
  title: 'My App',
  description: 'App description',
}

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en">
      <body className={inter.className}>
        {children}
      </body>
    </html>
  )
}
```

### Nested Layout
```tsx
// app/dashboard/layout.tsx
export default function DashboardLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <div className="flex">
      <Sidebar />
      <main className="flex-1">{children}</main>
    </div>
  )
}
```

## Data Fetching

### Server Component Fetching
```tsx
// No need for getServerSideProps!
async function getData() {
  const res = await fetch('https://api.example.com/data', {
    next: { revalidate: 3600 }, // ISR: revalidate every hour
    // Or: cache: 'no-store' for dynamic data
  })
  
  if (!res.ok) throw new Error('Failed to fetch')
  return res.json()
}

export default async function Page() {
  const data = await getData()
  return <div>{data.title}</div>
}
```

### Parallel Data Fetching
```tsx
export default async function Page() {
  // Start both fetches in parallel
  const postsPromise = getPosts()
  const usersPromise = getUsers()
  
  // Wait for both
  const [posts, users] = await Promise.all([postsPromise, usersPromise])
  
  return (
    <>
      <PostList posts={posts} />
      <UserList users={users} />
    </>
  )
}
```

### Streaming with Suspense
```tsx
import { Suspense } from 'react'

export default function Page() {
  return (
    <div>
      <h1>Dashboard</h1>
      
      {/* Streams in as data loads */}
      <Suspense fallback={<ChartSkeleton />}>
        <SlowChart />
      </Suspense>
      
      <Suspense fallback={<TableSkeleton />}>
        <SlowTable />
      </Suspense>
    </div>
  )
}
```

## Server Actions

```tsx
// app/actions.ts
'use server'

import { revalidatePath } from 'next/cache'
import { redirect } from 'next/navigation'
import { db } from '@/lib/db'

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string
  const content = formData.get('content') as string
  
  await db.post.create({
    data: { title, content },
  })
  
  revalidatePath('/posts')
  redirect('/posts')
}

export async function deletePost(id: string) {
  await db.post.delete({ where: { id } })
  revalidatePath('/posts')
}
```

### Using Server Actions
```tsx
// In a Server Component
import { createPost } from '@/app/actions'

export default function NewPostPage() {
  return (
    <form action={createPost}>
      <input name="title" required />
      <textarea name="content" required />
      <button type="submit">Create Post</button>
    </form>
  )
}
```

```tsx
// In a Client Component
'use client'

import { createPost } from '@/app/actions'
import { useTransition } from 'react'

export function CreatePostForm() {
  const [isPending, startTransition] = useTransition()
  
  async function handleSubmit(formData: FormData) {
    startTransition(async () => {
      await createPost(formData)
    })
  }
  
  return (
    <form action={handleSubmit}>
      <input name="title" disabled={isPending} />
      <button disabled={isPending}>
        {isPending ? 'Creating...' : 'Create'}
      </button>
    </form>
  )
}
```

## Route Handlers (API Routes)

```tsx
// app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server'

export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams
  const query = searchParams.get('q')
  
  const users = await db.user.findMany({
    where: query ? { name: { contains: query } } : undefined,
  })
  
  return NextResponse.json(users)
}

export async function POST(request: NextRequest) {
  const body = await request.json()
  
  const user = await db.user.create({ data: body })
  
  return NextResponse.json(user, { status: 201 })
}
```

### Dynamic Route Handlers
```tsx
// app/api/users/[id]/route.ts
export async function GET(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const user = await db.user.findUnique({
    where: { id: params.id },
  })
  
  if (!user) {
    return NextResponse.json({ error: 'Not found' }, { status: 404 })
  }
  
  return NextResponse.json(user)
}
```

## Middleware

```tsx
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  // Check auth
  const token = request.cookies.get('token')?.value
  
  if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url))
  }
  
  // Add headers
  const response = NextResponse.next()
  response.headers.set('x-pathname', request.nextUrl.pathname)
  
  return response
}

export const config = {
  matcher: [
    '/((?!api|_next/static|_next/image|favicon.ico).*)',
  ],
}
```

## Metadata & SEO

### Static Metadata
```tsx
// app/layout.tsx
import type { Metadata } from 'next'

export const metadata: Metadata = {
  title: {
    template: '%s | My App',
    default: 'My App',
  },
  description: 'App description',
  openGraph: {
    title: 'My App',
    description: 'App description',
    url: 'https://example.com',
    siteName: 'My App',
    images: ['/og.png'],
  },
  twitter: {
    card: 'summary_large_image',
    creator: '@myhandle',
  },
}
```

### Dynamic Metadata
```tsx
// app/posts/[slug]/page.tsx
import type { Metadata, ResolvingMetadata } from 'next'

type Props = {
  params: { slug: string }
}

export async function generateMetadata(
  { params }: Props,
  parent: ResolvingMetadata
): Promise<Metadata> {
  const post = await getPost(params.slug)
  
  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      images: [post.image],
    },
  }
}
```

## Error Handling

```tsx
// app/dashboard/error.tsx
'use client'

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  return (
    <div>
      <h2>Something went wrong!</h2>
      <p>{error.message}</p>
      <button onClick={() => reset()}>Try again</button>
    </div>
  )
}
```

```tsx
// app/not-found.tsx
export default function NotFound() {
  return (
    <div>
      <h2>Not Found</h2>
      <p>Could not find requested resource</p>
    </div>
  )
}
```

## Static & Dynamic Rendering

```tsx
// Force dynamic rendering
export const dynamic = 'force-dynamic'

// Or static with revalidation
export const revalidate = 3600 // seconds

// Generate static params for dynamic routes
export async function generateStaticParams() {
  const posts = await getPosts()
  return posts.map((post) => ({ slug: post.slug }))
}
```

## Best Practices

1. **Default to Server Components** - Only use 'use client' when needed
2. **Colocate client components** - Keep them as leaves in the tree
3. **Use Server Actions for mutations** - Type-safe, no API boilerplate
4. **Parallel fetch** - Use Promise.all for independent data
5. **Stream with Suspense** - Show content progressively
6. **Cache aggressively** - Use revalidate for ISR
7. **Preload data** - Use `preload` pattern for waterfalls
8. **Handle errors gracefully** - Add error.tsx at each route level
