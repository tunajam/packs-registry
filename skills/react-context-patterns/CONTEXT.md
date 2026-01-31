# React Context Patterns

Patterns for effective use of React Context API, including performance optimization and composition.

## Basic Context

```tsx
import { createContext, useContext, useState, ReactNode } from 'react'

// 1. Create context with type
interface ThemeContextType {
  theme: 'light' | 'dark'
  toggleTheme: () => void
}

const ThemeContext = createContext<ThemeContextType | null>(null)

// 2. Create provider
function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light')
  
  const toggleTheme = () => setTheme(t => t === 'light' ? 'dark' : 'light')
  
  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  )
}

// 3. Create custom hook
function useTheme() {
  const context = useContext(ThemeContext)
  if (!context) {
    throw new Error('useTheme must be used within ThemeProvider')
  }
  return context
}

// 4. Usage
function ThemeToggle() {
  const { theme, toggleTheme } = useTheme()
  return <button onClick={toggleTheme}>Current: {theme}</button>
}
```

## Separate State and Dispatch

Prevent unnecessary re-renders by splitting contexts:

```tsx
interface State {
  user: User | null
  isLoading: boolean
}

type Dispatch = React.Dispatch<Action>

const StateContext = createContext<State | null>(null)
const DispatchContext = createContext<Dispatch | null>(null)

function AuthProvider({ children }: { children: ReactNode }) {
  const [state, dispatch] = useReducer(authReducer, initialState)
  
  return (
    <StateContext.Provider value={state}>
      <DispatchContext.Provider value={dispatch}>
        {children}
      </DispatchContext.Provider>
    </StateContext.Provider>
  )
}

// Hooks
function useAuthState() {
  const context = useContext(StateContext)
  if (!context) throw new Error('useAuthState must be used within AuthProvider')
  return context
}

function useAuthDispatch() {
  const context = useContext(DispatchContext)
  if (!context) throw new Error('useAuthDispatch must be used within AuthProvider')
  return context
}

// Components that only dispatch don't re-render on state change
function LogoutButton() {
  const dispatch = useAuthDispatch() // Only dispatch, no state subscription
  return <button onClick={() => dispatch({ type: 'LOGOUT' })}>Logout</button>
}
```

## Context with Selectors

Use `use-context-selector` for fine-grained subscriptions:

```tsx
import { createContext, useContextSelector } from 'use-context-selector'

interface Store {
  count: number
  user: User | null
  increment: () => void
}

const StoreContext = createContext<Store | null>(null)

function StoreProvider({ children }: { children: ReactNode }) {
  const [count, setCount] = useState(0)
  const [user, setUser] = useState<User | null>(null)
  
  const value = useMemo(() => ({
    count,
    user,
    increment: () => setCount(c => c + 1),
  }), [count, user])
  
  return (
    <StoreContext.Provider value={value}>
      {children}
    </StoreContext.Provider>
  )
}

// Select only what you need
function Counter() {
  const count = useContextSelector(StoreContext, (s) => s?.count)
  const increment = useContextSelector(StoreContext, (s) => s?.increment)
  
  return <button onClick={increment}>{count}</button>
}

// This component won't re-render when count changes
function UserDisplay() {
  const user = useContextSelector(StoreContext, (s) => s?.user)
  return <div>{user?.name}</div>
}
```

## Compose Multiple Providers

```tsx
// Avoid provider hell
function Providers({ children }: { children: ReactNode }) {
  return (
    <AuthProvider>
      <ThemeProvider>
        <NotificationProvider>
          <CartProvider>
            {children}
          </CartProvider>
        </NotificationProvider>
      </ThemeProvider>
    </AuthProvider>
  )
}

// Or use a composer
function composeProviders(...providers: React.FC<{ children: ReactNode }>[]) {
  return ({ children }: { children: ReactNode }) =>
    providers.reduceRight(
      (acc, Provider) => <Provider>{acc}</Provider>,
      children
    )
}

const Providers = composeProviders(
  AuthProvider,
  ThemeProvider,
  NotificationProvider,
  CartProvider
)

// Usage
function App() {
  return (
    <Providers>
      <MainApp />
    </Providers>
  )
}
```

## Context with useReducer

For complex state logic:

```tsx
type State = {
  items: CartItem[]
  total: number
  discount: number
}

type Action =
  | { type: 'ADD_ITEM'; item: CartItem }
  | { type: 'REMOVE_ITEM'; id: string }
  | { type: 'APPLY_DISCOUNT'; code: string }
  | { type: 'CLEAR' }

function cartReducer(state: State, action: Action): State {
  switch (action.type) {
    case 'ADD_ITEM':
      return {
        ...state,
        items: [...state.items, action.item],
        total: state.total + action.item.price,
      }
    case 'REMOVE_ITEM':
      const item = state.items.find(i => i.id === action.id)
      return {
        ...state,
        items: state.items.filter(i => i.id !== action.id),
        total: state.total - (item?.price ?? 0),
      }
    case 'CLEAR':
      return { items: [], total: 0, discount: 0 }
    default:
      return state
  }
}

const CartContext = createContext<{
  state: State
  dispatch: React.Dispatch<Action>
} | null>(null)

function CartProvider({ children }: { children: ReactNode }) {
  const [state, dispatch] = useReducer(cartReducer, {
    items: [],
    total: 0,
    discount: 0,
  })

  return (
    <CartContext.Provider value={{ state, dispatch }}>
      {children}
    </CartContext.Provider>
  )
}
```

## Memoization for Performance

```tsx
function ExpensiveProvider({ children }: { children: ReactNode }) {
  const [count, setCount] = useState(0)
  
  // ❌ New object every render
  const value = { count, setCount }
  
  // ✅ Memoize the value
  const value = useMemo(() => ({ count, setCount }), [count])
  
  // ✅ Or memoize separately for actions
  const actions = useMemo(() => ({
    increment: () => setCount(c => c + 1),
    decrement: () => setCount(c => c - 1),
  }), [])
  
  return (
    <CountContext.Provider value={{ count, ...actions }}>
      {children}
    </CountContext.Provider>
  )
}
```

## Default Values Pattern

```tsx
// Option 1: Throw on missing provider (recommended)
const ThemeContext = createContext<Theme | null>(null)

function useTheme() {
  const context = useContext(ThemeContext)
  if (!context) {
    throw new Error('useTheme must be used within ThemeProvider')
  }
  return context
}

// Option 2: Provide sensible default
const ThemeContext = createContext<Theme>({
  theme: 'light',
  toggleTheme: () => {
    console.warn('ThemeProvider not found')
  },
})

// No null check needed, but missing provider is silent
function useTheme() {
  return useContext(ThemeContext)
}
```

## Lazy Context Initialization

```tsx
function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null)
  const [isLoading, setIsLoading] = useState(true)

  useEffect(() => {
    // Load user on mount
    getStoredUser()
      .then(setUser)
      .finally(() => setIsLoading(false))
  }, [])

  // Don't render children until initialized
  if (isLoading) {
    return <LoadingScreen />
  }

  return (
    <AuthContext.Provider value={{ user, setUser }}>
      {children}
    </AuthContext.Provider>
  )
}
```

## Context + Refs Pattern

For values that shouldn't trigger re-renders:

```tsx
function StableCallbackProvider({ children }: { children: ReactNode }) {
  const [state, setState] = useState(initialState)
  const stateRef = useRef(state)
  
  useEffect(() => {
    stateRef.current = state
  }, [state])
  
  // Stable callback that always has latest state
  const getState = useCallback(() => stateRef.current, [])
  
  const value = useMemo(() => ({
    state,
    getState, // Stable reference
    setState,
  }), [state, getState])
  
  return (
    <Context.Provider value={value}>
      {children}
    </Context.Provider>
  )
}
```

## Testing Context

```tsx
// Create test wrapper
function createTestWrapper(initialState?: Partial<State>) {
  return function Wrapper({ children }: { children: ReactNode }) {
    return (
      <StateProvider initialState={{ ...defaultState, ...initialState }}>
        {children}
      </StateProvider>
    )
  }
}

// Use in tests
describe('Component', () => {
  it('renders with context', () => {
    render(<Component />, {
      wrapper: createTestWrapper({ user: mockUser }),
    })
    // ...
  })
})
```

## Best Practices

1. **Always create custom hooks** - Hide context implementation
2. **Throw on missing provider** - Fail fast, clear errors
3. **Split state and dispatch** - Reduce unnecessary re-renders
4. **Memoize provider values** - Prevent cascading re-renders
5. **Consider alternatives** - Zustand/Jotai for complex state
6. **Don't overuse** - Props are fine for 1-2 levels
7. **Document your contexts** - What they provide, when to use
