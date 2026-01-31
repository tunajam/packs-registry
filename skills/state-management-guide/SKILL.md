# State Management Decision Guide

How to choose the right state management approach for your React application.

## State Categories

### 1. Local UI State

State that only one component needs.

```tsx
// ✅ Just use useState
function Counter() {
  const [count, setCount] = useState(0)
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>
}
```

**Examples:** Form inputs, toggles, local loading states

### 2. Shared UI State (Few Components)

State shared between a parent and its children.

```tsx
// ✅ Lift state up or use composition
function Parent() {
  const [selected, setSelected] = useState(null)
  return (
    <div>
      <Sidebar selected={selected} onSelect={setSelected} />
      <Content selected={selected} />
    </div>
  )
}
```

**Examples:** Selected item in a list, expanded/collapsed sections

### 3. Global UI State

State needed across distant components.

```tsx
// ✅ Use React Context or lightweight state manager (Zustand, Jotai)
const ThemeContext = createContext<Theme>('light')

// Or
const useThemeStore = create((set) => ({
  theme: 'light',
  toggle: () => set((s) => ({ theme: s.theme === 'light' ? 'dark' : 'light' }))
}))
```

**Examples:** Theme, user preferences, sidebar state

### 4. Server/Remote State

Data fetched from an API that needs caching/syncing.

```tsx
// ✅ Use TanStack Query or SWR
const { data, isLoading } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
})
```

**Examples:** User data, products, any API data

### 5. Complex UI Logic

Multi-step flows, state machines, complex transitions.

```tsx
// ✅ Use XState
const [state, send] = useMachine(checkoutMachine)
```

**Examples:** Checkout flows, multi-step forms, auth flows

## Decision Tree

```
Is it from a server/API?
├─ Yes → TanStack Query / SWR
└─ No → Is it complex with many states/transitions?
         ├─ Yes → XState
         └─ No → Is it needed in distant components?
                  ├─ Yes → How complex?
                  │         ├─ Simple → React Context
                  │         └─ Multiple slices → Zustand / Jotai
                  └─ No → Just props or useState
```

## Tool Comparison

| Tool | Best For | Trade-offs |
|------|----------|------------|
| **useState** | Local component state | None for its use case |
| **useReducer** | Complex local logic | Boilerplate |
| **Context** | Theme, auth, locale | Re-render concerns |
| **Zustand** | Global UI state | External dependency |
| **Jotai** | Atomic/fine-grained | Learning curve |
| **Valtio** | Mutable patterns | Proxy magic |
| **XState** | Complex flows | Verbosity |
| **TanStack Query** | Server state | Not for client state |
| **SWR** | Simple data fetching | Less features than Query |

## Context vs External State Manager

### Use Context When:

- Simple global state (theme, user, locale)
- State changes infrequently
- Don't need devtools/persistence
- Want to avoid external dependencies

```tsx
// Good for context
const UserContext = createContext<User | null>(null)

function UserProvider({ children }) {
  const [user, setUser] = useState<User | null>(null)
  return (
    <UserContext.Provider value={user}>
      {children}
    </UserContext.Provider>
  )
}
```

### Use External State Manager When:

- Complex state logic
- Need devtools for debugging
- Want persistence out of the box
- State changes frequently
- Need fine-grained subscriptions

```tsx
// Better with Zustand
const useStore = create(
  persist(
    devtools((set) => ({
      cart: [],
      addItem: (item) => set((s) => ({ cart: [...s.cart, item] })),
      removeItem: (id) => set((s) => ({ 
        cart: s.cart.filter(i => i.id !== id) 
      })),
    })),
    { name: 'cart-storage' }
  )
)
```

## Context Performance

Context causes re-renders for ALL consumers when value changes.

```tsx
// ❌ Problem: Everything re-renders on any change
const AppContext = createContext({ user: null, theme: 'light', cart: [] })

// ✅ Solution 1: Split contexts
const UserContext = createContext(null)
const ThemeContext = createContext('light')
const CartContext = createContext([])

// ✅ Solution 2: Use state manager with selectors
const useStore = create((set) => ({ user: null, theme: 'light', cart: [] }))

function Theme() {
  // Only re-renders when theme changes
  const theme = useStore((s) => s.theme)
}
```

## Common Patterns

### URL State

Search params are global, shareable state:

```tsx
// ✅ URL for filters, pagination, tabs
const [searchParams, setSearchParams] = useSearchParams()
const filter = searchParams.get('filter') ?? 'all'
const page = parseInt(searchParams.get('page') ?? '1')
```

### Form State

Forms have their own patterns:

```tsx
// ✅ Use form libraries for complex forms
import { useForm } from 'react-hook-form'

function Form() {
  const { register, handleSubmit } = useForm()
  return <form onSubmit={handleSubmit(onSubmit)}>...</form>
}

// ✅ Or simple useState for simple forms
function SimpleForm() {
  const [email, setEmail] = useState('')
  return <input value={email} onChange={e => setEmail(e.target.value)} />
}
```

### Hydration State

For SSR applications:

```tsx
// ✅ Handle hydration mismatch
function Component() {
  const [mounted, setMounted] = useState(false)
  useEffect(() => setMounted(true), [])
  
  if (!mounted) return <Skeleton />
  return <ClientOnlyContent />
}
```

## Anti-Patterns

### 1. Over-globalizing

```tsx
// ❌ Don't put everything in global state
const useStore = create((set) => ({
  modalOpen: false,  // Should be local
  inputValue: '',    // Should be local
  ...
}))

// ✅ Keep local state local
function Modal() {
  const [isOpen, setIsOpen] = useState(false)
}
```

### 2. Storing Derived Data

```tsx
// ❌ Don't store computed values
const useStore = create((set) => ({
  items: [],
  totalCount: 0,  // Derived!
  addItem: (item) => set((s) => ({
    items: [...s.items, item],
    totalCount: s.items.length + 1,
  })),
}))

// ✅ Compute on read
const useStore = create((set) => ({
  items: [],
  addItem: (item) => set((s) => ({ items: [...s.items, item] })),
}))

// In component
const items = useStore((s) => s.items)
const totalCount = items.length
```

### 3. Prop Drilling One Level

```tsx
// ❌ Don't use context to avoid one prop pass
<Parent>
  <Child data={data} />  // This is fine!
</Parent>

// ✅ Use context for truly distant components
<Root>
  <DeepNested>
    <Component />  // 10 levels deep, context makes sense
  </DeepNested>
</Root>
```

## Summary

1. **Start simple** - useState, lift state up
2. **Server data** - TanStack Query / SWR
3. **Global UI** - Context for simple, Zustand/Jotai for complex
4. **Complex flows** - XState
5. **URL state** - Search params for shareable state
6. **Don't over-engineer** - Match solution to problem size
