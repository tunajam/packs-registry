# Lazy Loading Routes

Code splitting and lazy loading patterns to improve initial load performance.

## React.lazy Basics

```tsx
import { lazy, Suspense } from 'react'

// Lazy import - creates a separate chunk
const Dashboard = lazy(() => import('./pages/Dashboard'))
const Settings = lazy(() => import('./pages/Settings'))

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  )
}
```

## Named Export Pattern

```tsx
// For named exports, wrap in default
const Dashboard = lazy(() =>
  import('./pages/Dashboard').then(module => ({
    default: module.Dashboard
  }))
)

// Or use a helper
function lazyNamed<T extends React.ComponentType<any>>(
  factory: () => Promise<{ [key: string]: T }>,
  name: string
) {
  return lazy(async () => {
    const module = await factory()
    return { default: module[name] }
  })
}

const Dashboard = lazyNamed(() => import('./pages/Dashboard'), 'Dashboard')
```

## React Router v7

File-based routing handles this automatically, but for manual routes:

```tsx
// routes/dashboard.tsx
import { lazy } from 'react'

// The component is lazy-loaded
export const handle = {
  // Can include preload hints
}

// This gets code-split automatically with file-based routing
export default function Dashboard() {
  return <h1>Dashboard</h1>
}
```

### clientLoader for Lazy Data

```tsx
// routes/products.$id.tsx
export async function clientLoader({ params }: Route.ClientLoaderArgs) {
  // Only loaded when this route is visited
  const [productModule, data] = await Promise.all([
    import('../features/products/ProductDetail'),
    fetch(`/api/products/${params.id}`).then(r => r.json()),
  ])
  
  return { data, Component: productModule.default }
}

export function HydrateFallback() {
  return <ProductSkeleton />
}
```

## TanStack Router

```tsx
// routes/dashboard.tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/dashboard')({
  component: () => import('./Dashboard').then(m => m.Dashboard),
  pendingComponent: () => <Loading />,
  errorComponent: ({ error }) => <Error error={error} />,
})
```

### With Preloading

```tsx
export const Route = createFileRoute('/dashboard')({
  // Load component lazily
  component: lazy(() => import('./Dashboard')),
  
  // Preload data
  loader: async () => {
    const data = await fetchDashboardData()
    return data
  },
  
  // Show while loading
  pendingComponent: () => <DashboardSkeleton />,
})
```

## Preload on Hover

```tsx
// Create preloadable lazy component
function preloadable<T extends React.ComponentType<any>>(
  factory: () => Promise<{ default: T }>
) {
  const Component = lazy(factory)
  
  return Object.assign(Component, {
    preload: factory,
  })
}

const Dashboard = preloadable(() => import('./pages/Dashboard'))

// Usage
<Link 
  to="/dashboard"
  onMouseEnter={() => Dashboard.preload()}
>
  Dashboard
</Link>
```

### With React Router

```tsx
import { Link } from 'react-router'

const routes = {
  dashboard: {
    path: '/dashboard',
    component: lazy(() => import('./pages/Dashboard')),
    preload: () => import('./pages/Dashboard'),
  },
}

function NavLink({ route }: { route: keyof typeof routes }) {
  const { path, preload } = routes[route]
  
  return (
    <Link 
      to={path}
      onMouseEnter={preload}
      onFocus={preload}
    >
      {route}
    </Link>
  )
}
```

## Route-Based Code Splitting

```tsx
// Group related components
const AdminModule = lazy(() => import('./modules/admin'))
const UserModule = lazy(() => import('./modules/user'))

function App() {
  return (
    <Suspense fallback={<FullPageLoader />}>
      <Routes>
        {/* User routes - one chunk */}
        <Route path="/user/*" element={<UserModule />} />
        
        {/* Admin routes - separate chunk */}
        <Route path="/admin/*" element={<AdminModule />} />
      </Routes>
    </Suspense>
  )
}

// modules/admin/index.tsx
export default function AdminModule() {
  return (
    <Routes>
      <Route path="/" element={<AdminDashboard />} />
      <Route path="/users" element={<AdminUsers />} />
      <Route path="/settings" element={<AdminSettings />} />
    </Routes>
  )
}
```

## Granular Suspense

```tsx
function App() {
  return (
    <Layout>
      {/* Different fallbacks for different sections */}
      <Suspense fallback={<SidebarSkeleton />}>
        <Sidebar />
      </Suspense>
      
      <main>
        <Suspense fallback={<ContentSkeleton />}>
          <Routes>
            <Route path="/dashboard" element={<Dashboard />} />
            <Route path="/settings" element={<Settings />} />
          </Routes>
        </Suspense>
      </main>
    </Layout>
  )
}
```

## Error Boundaries

```tsx
import { ErrorBoundary } from 'react-error-boundary'

function LazyRoutes() {
  return (
    <ErrorBoundary
      fallback={<div>Failed to load page</div>}
      onError={(error) => console.error('Chunk load failed:', error)}
    >
      <Suspense fallback={<Loading />}>
        <Routes>
          <Route path="/dashboard" element={<Dashboard />} />
        </Routes>
      </Suspense>
    </ErrorBoundary>
  )
}
```

## Retry Failed Chunks

```tsx
function lazyWithRetry<T extends React.ComponentType<any>>(
  componentImport: () => Promise<{ default: T }>,
  retries = 3,
  delay = 1000
): React.LazyExoticComponent<T> {
  return lazy(async () => {
    for (let i = 0; i < retries; i++) {
      try {
        return await componentImport()
      } catch (error) {
        if (i === retries - 1) throw error
        await new Promise(resolve => setTimeout(resolve, delay * (i + 1)))
      }
    }
    throw new Error('Failed to load after retries')
  })
}

const Dashboard = lazyWithRetry(() => import('./pages/Dashboard'))
```

## Prefetch Visible Routes

```tsx
import { useInView } from 'react-intersection-observer'

function PreloadableLink({ to, preload, children }: PreloadableLinkProps) {
  const { ref, inView } = useInView({ triggerOnce: true })
  
  useEffect(() => {
    if (inView) {
      preload()
    }
  }, [inView, preload])
  
  return (
    <Link ref={ref} to={to}>
      {children}
    </Link>
  )
}
```

## Vite Magic Comments

```tsx
// Named chunks
const Dashboard = lazy(() => 
  import(/* webpackChunkName: "dashboard" */ './pages/Dashboard')
)

// Prefetch hint (load during idle time)
const Settings = lazy(() =>
  import(/* webpackPrefetch: true */ './pages/Settings')
)

// Preload hint (load immediately in parallel)
const Profile = lazy(() =>
  import(/* webpackPreload: true */ './pages/Profile')
)
```

## Bundle Analysis

```bash
# Vite
npx vite-bundle-visualizer

# Webpack
npx webpack-bundle-analyzer
```

## Best Practices

1. **Split at route level** - Natural boundaries for chunks
2. **Preload on hover** - Reduce perceived latency
3. **Use granular Suspense** - Different skeletons per section
4. **Handle chunk failures** - Retry logic for network issues
5. **Analyze bundles** - Find optimization opportunities
6. **Group related features** - One chunk per feature/module
7. **Use named chunks** - Better debugging and caching
