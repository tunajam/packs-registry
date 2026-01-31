# React Router v7

React Router v7 is a framework for building React applications with file-based routing, data loading, and actions.

## Installation

```bash
npx create-react-router@latest my-app
cd my-app
npm run dev
```

## Project Structure

```
app/
├── routes/
│   ├── _index.tsx        # / (home)
│   ├── about.tsx         # /about
│   ├── products.tsx      # /products (layout)
│   ├── products._index.tsx # /products (index)
│   └── products.$id.tsx  # /products/:id
├── root.tsx
└── entry.client.tsx
```

## Route File Convention

```tsx
// routes/products.$id.tsx
import type { Route } from "./+types/products.$id"

// Data loading (runs on server during SSR)
export async function loader({ params }: Route.LoaderArgs) {
  const product = await db.products.find(params.id)
  if (!product) throw new Response("Not Found", { status: 404 })
  return product
}

// Component receives loader data
export default function Product({ loaderData }: Route.ComponentProps) {
  return (
    <div>
      <h1>{loaderData.name}</h1>
      <p>{loaderData.description}</p>
    </div>
  )
}
```

## Data Loading Patterns

### Server Loader (SSR)

```tsx
// Runs on server, removed from client bundle
export async function loader({ params }: Route.LoaderArgs) {
  const product = await fakeDb.getProduct(params.pid)
  return product
}
```

### Client Loader (SPA)

```tsx
// Runs in browser only
export async function clientLoader({ params }: Route.ClientLoaderArgs) {
  const res = await fetch(`/api/products/${params.pid}`)
  return res.json()
}

// Show while loading
export function HydrateFallback() {
  return <div>Loading...</div>
}
```

### Combined (SSR + Client)

```tsx
// Server for SSR, client for navigation
export async function loader({ params }: Route.LoaderArgs) {
  return fakeDb.getProduct(params.pid)
}

export async function clientLoader({ serverLoader, params }: Route.ClientLoaderArgs) {
  const res = await fetch(`/api/products/${params.pid}`)
  const serverData = await serverLoader()
  return { ...serverData, ...res.json() }
}
```

## Actions (Mutations)

```tsx
// routes/products.$id.tsx
export async function action({ request, params }: Route.ActionArgs) {
  const formData = await request.formData()
  const updates = Object.fromEntries(formData)
  
  await db.products.update(params.id, updates)
  
  return redirect(`/products/${params.id}`)
}

export default function EditProduct({ loaderData }: Route.ComponentProps) {
  return (
    <Form method="post">
      <input name="name" defaultValue={loaderData.name} />
      <button type="submit">Save</button>
    </Form>
  )
}
```

### Client Actions

```tsx
export async function clientAction({ request }: Route.ClientActionArgs) {
  const formData = await request.formData()
  await fetch('/api/products', {
    method: 'POST',
    body: formData,
  })
  return redirect('/products')
}
```

## Nested Routes

```tsx
// routes/dashboard.tsx (parent layout)
export default function Dashboard({ children }: { children: React.ReactNode }) {
  return (
    <div className="dashboard">
      <Sidebar />
      <main>
        <Outlet /> {/* Child routes render here */}
      </main>
    </div>
  )
}

// routes/dashboard.settings.tsx
export default function Settings() {
  return <h1>Settings</h1>
}
// Renders at /dashboard/settings inside Dashboard layout
```

## Navigation

```tsx
import { Link, NavLink, useNavigate } from 'react-router'

// Basic link
<Link to="/products">Products</Link>

// With params
<Link to={`/products/${product.id}`}>View</Link>

// Active styling
<NavLink 
  to="/about"
  className={({ isActive }) => isActive ? 'active' : ''}
>
  About
</NavLink>

// Programmatic navigation
function Component() {
  const navigate = useNavigate()
  
  const handleClick = () => {
    navigate('/products')
    // or navigate(-1) to go back
  }
}
```

## URL Parameters

```tsx
// Route: /products/:id
import { useParams } from 'react-router'

function Product() {
  const { id } = useParams()
  return <div>Product {id}</div>
}

// Search params
import { useSearchParams } from 'react-router'

function Products() {
  const [searchParams, setSearchParams] = useSearchParams()
  const filter = searchParams.get('filter')
  
  return (
    <button onClick={() => setSearchParams({ filter: 'new' })}>
      Filter New
    </button>
  )
}
```

## Error Handling

```tsx
// routes/products.$id.tsx
export function ErrorBoundary({ error }: Route.ErrorBoundaryProps) {
  if (error instanceof Response && error.status === 404) {
    return <h1>Product not found</h1>
  }
  
  return (
    <div>
      <h1>Error</h1>
      <p>{error.message}</p>
    </div>
  )
}
```

## Pending UI

```tsx
import { useNavigation } from 'react-router'

function Layout() {
  const navigation = useNavigation()
  
  return (
    <div>
      {navigation.state === 'loading' && <GlobalSpinner />}
      <Outlet />
    </div>
  )
}
```

## Fetchers (Non-Navigation Mutations)

```tsx
import { useFetcher } from 'react-router'

function AddToCart({ productId }: { productId: string }) {
  const fetcher = useFetcher()
  
  return (
    <fetcher.Form method="post" action="/cart">
      <input type="hidden" name="productId" value={productId} />
      <button type="submit" disabled={fetcher.state !== 'idle'}>
        {fetcher.state === 'idle' ? 'Add to Cart' : 'Adding...'}
      </button>
    </fetcher.Form>
  )
}
```

## Pre-rendering (Static)

```tsx
// react-router.config.ts
import type { Config } from "@react-router/dev/config"

export default {
  async prerender() {
    const products = await getProducts()
    return [
      '/',
      '/about',
      ...products.map(p => `/products/${p.id}`)
    ]
  },
} satisfies Config
```

## Protected Routes

```tsx
// routes/dashboard.tsx
export async function loader({ request }: Route.LoaderArgs) {
  const user = await getUser(request)
  if (!user) {
    throw redirect('/login')
  }
  return user
}
```

## Layout Routes (Pathless)

```tsx
// routes/_auth.tsx (layout without path segment)
export default function AuthLayout() {
  return (
    <div className="auth-container">
      <Outlet />
    </div>
  )
}

// routes/_auth.login.tsx -> /login
// routes/_auth.register.tsx -> /register
// Both use AuthLayout without adding to URL
```

## Type Safety

```tsx
// Types are auto-generated from route file
import type { Route } from "./+types/products.$id"

export async function loader({ params }: Route.LoaderArgs) {
  // params.id is typed
  return db.products.find(params.id)
}

export default function Product({ loaderData }: Route.ComponentProps) {
  // loaderData is fully typed from loader return
  return <h1>{loaderData.name}</h1>
}
```

## Best Practices

1. **Use loaders for data** - Fetch in loaders, not useEffect
2. **Actions for mutations** - Use Form + action, not fetch
3. **Leverage nested routes** - Share layouts efficiently
4. **Handle errors at route level** - ErrorBoundary per route
5. **Use fetchers for non-navigation** - Updates without navigation
6. **Prefetch on hover** - `<Link prefetch="intent">`
7. **Leverage type generation** - Import from +types
