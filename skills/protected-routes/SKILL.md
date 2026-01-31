# Protected Routes

Patterns for implementing authentication-protected routes in React applications.

## React Router v7 Pattern

### Using Loaders

```tsx
// routes/dashboard.tsx
import { redirect } from 'react-router'
import type { Route } from './+types/dashboard'

export async function loader({ request }: Route.LoaderArgs) {
  const user = await getUser(request) // Check session/token
  
  if (!user) {
    // Redirect to login with return URL
    const url = new URL(request.url)
    throw redirect(`/login?redirectTo=${url.pathname}`)
  }
  
  return { user }
}

export default function Dashboard({ loaderData }: Route.ComponentProps) {
  return <h1>Welcome, {loaderData.user.name}</h1>
}
```

### Root Layout Protection

```tsx
// routes/_protected.tsx (layout route)
export async function loader({ request }: Route.LoaderArgs) {
  const user = await getUser(request)
  if (!user) throw redirect('/login')
  return { user }
}

export default function ProtectedLayout() {
  return (
    <div>
      <Sidebar />
      <Outlet /> {/* All child routes are protected */}
    </div>
  )
}

// routes/_protected.dashboard.tsx -> /dashboard (protected)
// routes/_protected.settings.tsx -> /settings (protected)
```

### Login Handler

```tsx
// routes/login.tsx
import { Form, redirect, useSearchParams } from 'react-router'

export async function action({ request }: Route.ActionArgs) {
  const formData = await request.formData()
  const email = formData.get('email')
  const password = formData.get('password')
  
  const user = await login(email, password)
  if (!user) {
    return { error: 'Invalid credentials' }
  }
  
  // Get redirect URL from search params
  const url = new URL(request.url)
  const redirectTo = url.searchParams.get('redirectTo') || '/dashboard'
  
  return redirect(redirectTo, {
    headers: {
      'Set-Cookie': await createSession(user),
    },
  })
}

export default function Login({ actionData }: Route.ComponentProps) {
  const [searchParams] = useSearchParams()
  
  return (
    <Form method="post">
      {actionData?.error && <p className="error">{actionData.error}</p>}
      <input name="email" type="email" required />
      <input name="password" type="password" required />
      <button type="submit">Login</button>
    </Form>
  )
}
```

## TanStack Router Pattern

### Route Context

```tsx
// routes/__root.tsx
import { createRootRouteWithContext } from '@tanstack/react-router'

interface RouterContext {
  auth: {
    user: User | null
    isAuthenticated: boolean
  }
}

export const Route = createRootRouteWithContext<RouterContext>()({
  component: RootComponent,
})
```

### Protected Layout

```tsx
// routes/_authenticated.tsx
import { createFileRoute, redirect } from '@tanstack/react-router'

export const Route = createFileRoute('/_authenticated')({
  beforeLoad: async ({ context, location }) => {
    if (!context.auth.isAuthenticated) {
      throw redirect({
        to: '/login',
        search: {
          redirect: location.href,
        },
      })
    }
  },
  component: AuthenticatedLayout,
})

function AuthenticatedLayout() {
  return (
    <div>
      <nav>Protected Navigation</nav>
      <Outlet />
    </div>
  )
}

// routes/_authenticated/dashboard.tsx -> /dashboard
// routes/_authenticated/profile.tsx -> /profile
```

### Auth Context Provider

```tsx
// main.tsx
import { RouterProvider, createRouter } from '@tanstack/react-router'
import { routeTree } from './routeTree.gen'

function App() {
  const auth = useAuth() // Your auth hook
  
  const router = createRouter({
    routeTree,
    context: {
      auth: {
        user: auth.user,
        isAuthenticated: !!auth.user,
      },
    },
  })
  
  return <RouterProvider router={router} />
}
```

## Component-Based Pattern

For simpler apps or when not using file-based routing:

```tsx
function ProtectedRoute({ children }: { children: React.ReactNode }) {
  const { user, isLoading } = useAuth()
  const location = useLocation()
  
  if (isLoading) {
    return <LoadingSpinner />
  }
  
  if (!user) {
    return <Navigate to="/login" state={{ from: location }} replace />
  }
  
  return <>{children}</>
}

// Usage
<Routes>
  <Route path="/login" element={<Login />} />
  <Route
    path="/dashboard"
    element={
      <ProtectedRoute>
        <Dashboard />
      </ProtectedRoute>
    }
  />
</Routes>
```

## Role-Based Access

```tsx
// React Router v7
export async function loader({ request }: Route.LoaderArgs) {
  const user = await getUser(request)
  
  if (!user) {
    throw redirect('/login')
  }
  
  if (!user.roles.includes('admin')) {
    throw redirect('/unauthorized')
  }
  
  return { user }
}

// TanStack Router
export const Route = createFileRoute('/admin')({
  beforeLoad: async ({ context }) => {
    if (!context.auth.user?.roles.includes('admin')) {
      throw redirect({ to: '/unauthorized' })
    }
  },
})

// Component pattern
function RequireRole({ role, children }: { role: string; children: ReactNode }) {
  const { user } = useAuth()
  
  if (!user?.roles.includes(role)) {
    return <Navigate to="/unauthorized" />
  }
  
  return <>{children}</>
}
```

## Handling Token Expiry

```tsx
// Create an auth-aware fetch
async function authFetch(url: string, options?: RequestInit) {
  const token = getToken()
  
  const response = await fetch(url, {
    ...options,
    headers: {
      ...options?.headers,
      Authorization: `Bearer ${token}`,
    },
  })
  
  if (response.status === 401) {
    // Token expired, redirect to login
    clearToken()
    window.location.href = `/login?redirect=${window.location.pathname}`
    throw new Error('Session expired')
  }
  
  return response
}

// Use in loaders
export async function loader({ request }: Route.LoaderArgs) {
  try {
    const data = await authFetch('/api/protected-data')
    return data.json()
  } catch (error) {
    throw redirect('/login')
  }
}
```

## Preserving State on Login

```tsx
// Store attempted URL
function Login() {
  const [searchParams] = useSearchParams()
  const location = useLocation()
  const redirectTo = searchParams.get('redirect') || location.state?.from?.pathname || '/'
  
  async function handleLogin(credentials: Credentials) {
    await login(credentials)
    navigate(redirectTo, { replace: true })
  }
}

// In protected route
function ProtectedRoute({ children }: { children: ReactNode }) {
  const { user } = useAuth()
  const location = useLocation()
  
  if (!user) {
    return (
      <Navigate 
        to="/login" 
        state={{ from: location }}  // Pass current location
        replace 
      />
    )
  }
  
  return <>{children}</>
}
```

## SSR Considerations

```tsx
// React Router v7 with cookies
export async function loader({ request }: Route.LoaderArgs) {
  const cookieHeader = request.headers.get('Cookie')
  const session = await getSession(cookieHeader)
  
  if (!session.has('userId')) {
    throw redirect('/login')
  }
  
  return { userId: session.get('userId') }
}

// TanStack Router with SSR
export const Route = createFileRoute('/dashboard')({
  beforeLoad: async ({ context }) => {
    // context.auth should be populated server-side
    if (!context.auth.isAuthenticated) {
      throw redirect({ to: '/login' })
    }
  },
})
```

## Best Practices

1. **Use route-level protection** - Loaders/beforeLoad, not components
2. **Preserve redirect URL** - Return users to where they were going
3. **Handle loading states** - Show spinner during auth check
4. **Layout routes for groups** - Protect many routes at once
5. **Server-side validation** - Don't rely solely on client checks
6. **Clear tokens on 401** - Handle session expiry gracefully
7. **Role-based for fine-grained** - Admin vs user routes
