# TanStack Router

100% type-safe router with first-class search params, built-in caching, and file-based routing.

## Installation

```bash
npm install @tanstack/react-router
npm install -D @tanstack/router-plugin @tanstack/router-devtools
```

## Vite Setup

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import { TanStackRouterVite } from '@tanstack/router-plugin/vite'

export default defineConfig({
  plugins: [TanStackRouterVite(), react()],
})
```

## File-Based Routing

```
src/
├── routes/
│   ├── __root.tsx           # Root layout
│   ├── index.tsx            # /
│   ├── about.tsx            # /about
│   ├── products/
│   │   ├── index.tsx        # /products
│   │   └── $productId.tsx   # /products/:productId
│   └── _auth/               # Pathless layout group
│       ├── login.tsx        # /login
│       └── register.tsx     # /register
└── routeTree.gen.ts         # Auto-generated
```

## Root Route

```tsx
// routes/__root.tsx
import { Outlet, createRootRoute } from '@tanstack/react-router'

export const Route = createRootRoute({
  component: RootComponent,
})

function RootComponent() {
  return (
    <>
      <nav>
        <Link to="/">Home</Link>
        <Link to="/about">About</Link>
      </nav>
      <Outlet />
    </>
  )
}
```

## Basic Route

```tsx
// routes/about.tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/about')({
  component: AboutPage,
})

function AboutPage() {
  return <h1>About Us</h1>
}
```

## Route Parameters

```tsx
// routes/products/$productId.tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/products/$productId')({
  loader: async ({ params }) => {
    // params.productId is fully typed!
    return fetchProduct(params.productId)
  },
  component: ProductPage,
})

function ProductPage() {
  const product = Route.useLoaderData()
  const { productId } = Route.useParams()
  
  return <h1>{product.name}</h1>
}
```

## Search Parameters (First-Class!)

```tsx
// routes/products/index.tsx
import { createFileRoute } from '@tanstack/react-router'
import { z } from 'zod'

// Validate and type search params
const productSearchSchema = z.object({
  page: z.number().default(1),
  filter: z.enum(['all', 'active', 'archived']).default('all'),
  search: z.string().optional(),
})

export const Route = createFileRoute('/products/')({
  validateSearch: productSearchSchema,
  component: ProductsPage,
})

function ProductsPage() {
  // search is fully typed!
  const { page, filter, search } = Route.useSearch()
  const navigate = Route.useNavigate()
  
  return (
    <div>
      <input
        value={search ?? ''}
        onChange={(e) => 
          navigate({ search: { search: e.target.value } })
        }
      />
      <button onClick={() => navigate({ search: { page: page + 1 } })}>
        Next Page
      </button>
    </div>
  )
}
```

## Navigation

```tsx
import { Link, useNavigate } from '@tanstack/react-router'

// Type-safe links
<Link to="/products/$productId" params={{ productId: '123' }}>
  View Product
</Link>

// With search params
<Link
  to="/products"
  search={{ filter: 'active', page: 1 }}
>
  Active Products
</Link>

// Programmatic
function Component() {
  const navigate = useNavigate()
  
  const handleClick = () => {
    navigate({
      to: '/products/$productId',
      params: { productId: '123' },
      search: { tab: 'details' },
    })
  }
}
```

## Data Loading

```tsx
// routes/products/$productId.tsx
export const Route = createFileRoute('/products/$productId')({
  loader: async ({ params, context }) => {
    const product = await fetchProduct(params.productId)
    return { product }
  },
  component: ProductPage,
})

function ProductPage() {
  const { product } = Route.useLoaderData()
  return <h1>{product.name}</h1>
}
```

### With TanStack Query

```tsx
export const Route = createFileRoute('/products/$productId')({
  loader: async ({ params, context }) => {
    // Use queryClient from context
    await context.queryClient.ensureQueryData({
      queryKey: ['product', params.productId],
      queryFn: () => fetchProduct(params.productId),
    })
  },
  component: ProductPage,
})

function ProductPage() {
  const { productId } = Route.useParams()
  const { data: product } = useSuspenseQuery({
    queryKey: ['product', productId],
    queryFn: () => fetchProduct(productId),
  })
  
  return <h1>{product.name}</h1>
}
```

## Route Context

```tsx
// routes/__root.tsx
export const Route = createRootRoute({
  beforeLoad: async () => {
    return {
      user: await getUser(),
      queryClient: new QueryClient(),
    }
  },
})

// Child routes can access context
// routes/dashboard.tsx
export const Route = createFileRoute('/dashboard')({
  beforeLoad: async ({ context }) => {
    if (!context.user) {
      throw redirect({ to: '/login' })
    }
  },
})
```

## Preloading

```tsx
// Preload on hover (default)
<Link to="/products/$productId" params={{ productId: '123' }} preload="intent">
  View Product
</Link>

// Preload on render
<Link preload="render">...</Link>

// Programmatic
const preloadProduct = Route.usePreload()
<button onMouseEnter={() => preloadProduct({ productId: '123' })}>
  View
</button>
```

## Error Handling

```tsx
export const Route = createFileRoute('/products/$productId')({
  loader: async ({ params }) => {
    const product = await fetchProduct(params.productId)
    if (!product) {
      throw new Error('Product not found')
    }
    return product
  },
  errorComponent: ({ error }) => {
    return <div>Error: {error.message}</div>
  },
  pendingComponent: () => <Spinner />,
  component: ProductPage,
})
```

## Not Found Routes

```tsx
// routes/__root.tsx
export const Route = createRootRoute({
  notFoundComponent: () => <h1>404 Not Found</h1>,
})

// Or throw notFound() in loaders
export const Route = createFileRoute('/products/$productId')({
  loader: async ({ params }) => {
    const product = await fetchProduct(params.productId)
    if (!product) {
      throw notFound()
    }
    return product
  },
})
```

## Layout Routes

```tsx
// routes/_authenticated.tsx (pathless layout)
export const Route = createFileRoute('/_authenticated')({
  beforeLoad: async ({ context }) => {
    if (!context.user) {
      throw redirect({ to: '/login' })
    }
  },
  component: AuthenticatedLayout,
})

function AuthenticatedLayout() {
  return (
    <div>
      <Sidebar />
      <Outlet />
    </div>
  )
}

// routes/_authenticated/dashboard.tsx -> /dashboard
// routes/_authenticated/settings.tsx -> /settings
```

## Search Param Middleware

```tsx
export const Route = createFileRoute('/products/')({
  validateSearch: productSearchSchema,
  search: {
    middlewares: [
      // Strip defaults from URL
      ({ search, next }) => {
        const result = next(search)
        if (result.page === 1) delete result.page
        return result
      },
    ],
  },
})
```

## DevTools

```tsx
import { TanStackRouterDevtools } from '@tanstack/router-devtools'

function RootComponent() {
  return (
    <>
      <Outlet />
      <TanStackRouterDevtools />
    </>
  )
}
```

## Type-Safe Links

```tsx
// TypeScript ensures valid routes and params
<Link
  to="/products/$productId"
  params={{ productId: '123' }}  // Required, typed
  search={{ tab: 'details' }}    // Validated against route schema
>
  View
</Link>

// Errors at compile time:
<Link to="/invalid-route" />  // ❌ Route doesn't exist
<Link to="/products/$productId" /> // ❌ Missing params
```

## Best Practices

1. **Use search params for UI state** - Bookmarkable, shareable
2. **Validate search with zod** - Type-safe search params
3. **Preload on intent** - Better perceived performance
4. **Use context for shared data** - Auth, queryClient
5. **Leverage file-based routing** - Auto-generated route tree
6. **Use suspense with loaders** - Simpler loading states
7. **Enable devtools** - Debug routing issues
