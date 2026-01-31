# Valtio Proxy State

Proxy state made simple. Valtio's proxy turns objects into self-aware proxies, allowing fine-grained subscription and reactivity. Shines at render optimization in React.

## Installation

```bash
npm install valtio
```

## Core Concepts

### Proxy & Snapshot

```tsx
import { proxy, useSnapshot } from 'valtio'

// Create proxy state
const state = proxy({
  count: 0,
  users: [] as User[],
})

// Read with useSnapshot (reactive)
function Counter() {
  const snap = useSnapshot(state)
  return <div>{snap.count}</div>
}

// Write by mutating the proxy directly
function Controls() {
  return (
    <button onClick={() => state.count++}>
      Increment
    </button>
  )
}
```

### Key Insight: Proxy vs Snapshot

- **proxy**: Mutable, for writing. Mutate this.
- **snapshot**: Immutable, for reading. Renders use this.

```tsx
// ✅ Correct - mutate proxy
state.count++
state.users.push(newUser)

// ❌ Wrong - don't mutate snapshot
const snap = useSnapshot(state)
snap.count++ // Error! Snapshots are readonly
```

## Actions Pattern

```tsx
const store = proxy({
  todos: [] as Todo[],
  filter: 'all' as 'all' | 'active' | 'completed',
})

// Actions - just functions that mutate the proxy
const actions = {
  addTodo: (text: string) => {
    store.todos.push({
      id: Date.now(),
      text,
      completed: false,
    })
  },
  
  toggleTodo: (id: number) => {
    const todo = store.todos.find(t => t.id === id)
    if (todo) todo.completed = !todo.completed
  },
  
  removeTodo: (id: number) => {
    const index = store.todos.findIndex(t => t.id === id)
    if (index >= 0) store.todos.splice(index, 1)
  },
  
  setFilter: (filter: typeof store.filter) => {
    store.filter = filter
  },
}

export { store, actions }
```

## Derived State

```tsx
import { proxy, useSnapshot } from 'valtio'
import { derive } from 'valtio/utils'

const state = proxy({
  count: 1,
})

// Derived values (computed)
const derived = derive({
  doubled: (get) => get(state).count * 2,
  quadrupled: (get) => get(state).count * 4,
})

function Display() {
  const snap = useSnapshot(derived)
  return <div>Doubled: {snap.doubled}</div>
}
```

## Subscriptions

### Subscribe to Changes

```tsx
import { subscribe } from 'valtio'

// Subscribe to any change
const unsubscribe = subscribe(state, () => {
  console.log('State changed:', state)
})

// Subscribe to specific path
import { subscribeKey } from 'valtio/utils'

subscribeKey(state, 'count', (value) => {
  console.log('Count is now:', value)
})
```

### Watch (Selective Subscription)

```tsx
import { watch } from 'valtio/utils'

// Only triggers when the getter result changes
const stop = watch((get) => {
  const count = get(state).count
  console.log('Count changed to:', count)
})
```

## Ref (Escape Hatch)

```tsx
import { proxy, ref } from 'valtio'

const state = proxy({
  count: 0,
  // ref() prevents deep proxy wrapping
  domElement: ref(document.createElement('div')),
  // Useful for non-serializable values
  websocket: ref(new WebSocket('ws://...')),
})
```

## Persistence

### localStorage

```tsx
import { proxy, subscribe } from 'valtio'

const state = proxy(
  JSON.parse(localStorage.getItem('app-state') || '{}')
)

subscribe(state, () => {
  localStorage.setItem('app-state', JSON.stringify(state))
})
```

### proxyWithPersist Utility

```tsx
import { proxyWithPersist } from 'valtio-persist'

const state = proxyWithPersist({
  name: 'app-state',
  initialState: { count: 0 },
  storage: localStorage,
})
```

## Nested Proxy Access

```tsx
const state = proxy({
  users: [
    { id: 1, name: 'Alice', settings: { theme: 'dark' } }
  ],
})

// Pass nested proxy to useSnapshot for fine-grained updates
function UserSettings({ index }: { index: number }) {
  // Only re-renders when this specific user's settings change
  const snap = useSnapshot(state.users[index].settings)
  return <div>Theme: {snap.theme}</div>
}
```

## Module Scope Mutations

Valtio excels at state updates outside React:

```tsx
// state.ts
export const state = proxy({ count: 0 })

// api.ts - mutate from anywhere
import { state } from './state'

export async function fetchAndUpdateCount() {
  const data = await fetch('/api/count').then(r => r.json())
  state.count = data.count // Works outside React!
}

// Timer example
setInterval(() => {
  state.count++
}, 1000)
```

## TypeScript Patterns

```tsx
interface AppState {
  user: User | null
  todos: Todo[]
  settings: Settings
}

const state = proxy<AppState>({
  user: null,
  todos: [],
  settings: { theme: 'light' },
})

// Typed actions
const actions = {
  setUser: (user: User | null) => {
    state.user = user
  },
  addTodo: (todo: Omit<Todo, 'id'>) => {
    state.todos.push({ ...todo, id: crypto.randomUUID() })
  },
}
```

## DevTools

```tsx
import { devtools } from 'valtio/utils'

const state = proxy({ count: 0 })
devtools(state, { name: 'app-state', enabled: true })
```

## Best Practices

1. **Mutate proxy, read snapshot** - Never confuse which to use
2. **Keep proxy at module scope** - Enables mutations from anywhere
3. **Use actions for mutations** - Easier to track and debug
4. **Fine-grained snapshots** - Pass nested proxies to useSnapshot
5. **ref() for non-proxied values** - DOM refs, WebSockets, etc.
6. **derive() for computed state** - Instead of storing derived values

## When to Use Valtio

- You prefer mutable patterns over immutable
- Need to update state outside React (timers, websockets)
- Want fine-grained reactivity without selectors
- Coming from Vue or MobX backgrounds
