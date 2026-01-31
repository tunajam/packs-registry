# Jotai Atoms

Primitive and flexible state management for React. Jotai's atomic approach scales from a simple useState replacement to enterprise applications.

## Installation

```bash
npm install jotai
```

## Core Concepts

### Basic Atoms

```tsx
import { atom, useAtom } from 'jotai'

// Primitive atom
const countAtom = atom(0)

// Component
function Counter() {
  const [count, setCount] = useAtom(countAtom)
  return (
    <button onClick={() => setCount(c => c + 1)}>
      Count: {count}
    </button>
  )
}
```

### Read-Only Derived Atoms

```tsx
const countAtom = atom(0)
const doubleCountAtom = atom((get) => get(countAtom) * 2)

function Display() {
  const [doubleCount] = useAtom(doubleCountAtom)
  return <div>Double: {doubleCount}</div>
}
```

### Write-Only Atoms

```tsx
const countAtom = atom(0)

const incrementAtom = atom(
  null, // read function returns null
  (get, set) => set(countAtom, get(countAtom) + 1)
)

function Button() {
  const [, increment] = useAtom(incrementAtom)
  return <button onClick={increment}>+1</button>
}
```

### Read-Write Derived Atoms

```tsx
const celsiusAtom = atom(0)

const fahrenheitAtom = atom(
  (get) => get(celsiusAtom) * 9/5 + 32,
  (get, set, newValue: number) => set(celsiusAtom, (newValue - 32) * 5/9)
)
```

## Async Atoms

### Async Read

```tsx
const userAtom = atom(async (get) => {
  const userId = get(userIdAtom)
  const response = await fetch(`/api/users/${userId}`)
  return response.json()
})

// With Suspense
function UserProfile() {
  const [user] = useAtom(userAtom)
  return <div>{user.name}</div>
}
```

### Async Write

```tsx
const doSomethingAtom = atom(
  null,
  async (get, set, payload) => {
    const result = await asyncOperation(payload)
    set(resultAtom, result)
  }
)
```

### Loadable (No Suspense)

```tsx
import { loadable } from 'jotai/utils'

const asyncAtom = atom(async () => fetch('/api/data').then(r => r.json()))
const loadableAtom = loadable(asyncAtom)

function Component() {
  const [value] = useAtom(loadableAtom)
  
  if (value.state === 'loading') return <Loading />
  if (value.state === 'hasError') return <Error error={value.error} />
  return <Data data={value.data} />
}
```

## Utilities

### atomWithStorage (Persistence)

```tsx
import { atomWithStorage } from 'jotai/utils'

// Persists to localStorage
const themeAtom = atomWithStorage('theme', 'light')

// Cross-tab sync included by default
const settingsAtom = atomWithStorage('settings', { 
  notifications: true 
})

// Custom storage (sessionStorage)
import { createJSONStorage } from 'jotai/utils'

const sessionAtom = atomWithStorage(
  'session',
  null,
  createJSONStorage(() => sessionStorage)
)
```

### atomFamily (Dynamic Atoms)

```tsx
import { atomFamily } from 'jotai/utils'

const todoAtomFamily = atomFamily((id: string) => 
  atom({ id, text: '', completed: false })
)

// Usage - each id gets its own atom instance
function Todo({ id }: { id: string }) {
  const [todo, setTodo] = useAtom(todoAtomFamily(id))
  return <div>{todo.text}</div>
}
```

### atomWithReset

```tsx
import { atomWithReset, useResetAtom } from 'jotai/utils'

const countAtom = atomWithReset(0)

function Counter() {
  const [count, setCount] = useAtom(countAtom)
  const reset = useResetAtom(countAtom)
  
  return (
    <>
      <span>{count}</span>
      <button onClick={() => setCount(c => c + 1)}>+</button>
      <button onClick={reset}>Reset</button>
    </>
  )
}
```

### atomWithReducer

```tsx
import { atomWithReducer } from 'jotai/utils'

type Action = { type: 'inc' } | { type: 'dec' } | { type: 'set', value: number }

const countAtom = atomWithReducer(0, (state, action: Action) => {
  switch (action.type) {
    case 'inc': return state + 1
    case 'dec': return state - 1
    case 'set': return action.value
  }
})

function Counter() {
  const [count, dispatch] = useAtom(countAtom)
  return (
    <>
      <span>{count}</span>
      <button onClick={() => dispatch({ type: 'inc' })}>+</button>
    </>
  )
}
```

### selectAtom (Fine-grained Subscriptions)

```tsx
import { selectAtom } from 'jotai/utils'

const personAtom = atom({ name: 'Alice', age: 30 })

// Only re-renders when name changes
const nameAtom = selectAtom(personAtom, (person) => person.name)

// With equality function
const ageAtom = selectAtom(
  personAtom,
  (person) => person.age,
  (a, b) => a === b
)
```

## Extensions

### Jotai + TanStack Query

```tsx
import { atomWithQuery, atomWithMutation } from 'jotai-tanstack-query'

const userAtom = atomWithQuery((get) => ({
  queryKey: ['user', get(userIdAtom)],
  queryFn: async ({ queryKey: [, userId] }) => {
    const res = await fetch(`/api/users/${userId}`)
    return res.json()
  },
}))

const updateUserAtom = atomWithMutation(() => ({
  mutationFn: async (user: User) => {
    await fetch('/api/users', { 
      method: 'POST', 
      body: JSON.stringify(user) 
    })
  },
}))
```

### Jotai + Immer

```tsx
import { atomWithImmer } from 'jotai-immer'

const todosAtom = atomWithImmer([
  { id: 1, text: 'Learn Jotai', done: false }
])

function Todos() {
  const [todos, setTodos] = useAtom(todosAtom)
  
  const toggle = (id: number) => {
    setTodos(draft => {
      const todo = draft.find(t => t.id === id)
      if (todo) todo.done = !todo.done
    })
  }
}
```

### Jotai + XState

```tsx
import { atomWithMachine } from 'jotai-xstate'
import { createMachine } from 'xstate'

const toggleMachine = createMachine({
  id: 'toggle',
  initial: 'inactive',
  states: {
    inactive: { on: { TOGGLE: 'active' } },
    active: { on: { TOGGLE: 'inactive' } },
  },
})

const toggleAtom = atomWithMachine(toggleMachine)

function Toggle() {
  const [state, send] = useAtom(toggleAtom)
  return (
    <button onClick={() => send({ type: 'TOGGLE' })}>
      {state.value}
    </button>
  )
}
```

## Provider (Optional Scoping)

```tsx
import { Provider, createStore } from 'jotai'

// Create isolated store
const myStore = createStore()

// Set initial values
myStore.set(countAtom, 10)

function App() {
  return (
    <Provider store={myStore}>
      <Counter />
    </Provider>
  )
}
```

## DevTools

```tsx
import { useAtomsDevtools } from 'jotai-devtools'

function AtomsDevtools({ children }) {
  useAtomsDevtools('my-app')
  return children
}
```

## Best Practices

1. **Start small** - Single atoms for simple state, compose for complex
2. **Derive don't duplicate** - Use derived atoms instead of storing computed values
3. **Keep atoms focused** - One piece of state per atom
4. **Use atom families for dynamic data** - Lists, entities by ID
5. **Prefer atomWithStorage for persistence** - Built-in cross-tab sync
6. **Use selectAtom for large objects** - Prevent unnecessary re-renders
