# Zustand Patterns

A small, fast, and scalable bearbones state management solution. Zustand has a comfy API based on hooks — not boilerplatey or opinionated, but has enough convention to be explicit and flux-like.

## Installation

```bash
npm install zustand
```

## Core Patterns

### Basic Store

```tsx
import { create } from 'zustand'

interface BearState {
  bears: number
  increase: () => void
  decrease: () => void
  reset: () => void
}

const useBearStore = create<BearState>((set) => ({
  bears: 0,
  increase: () => set((state) => ({ bears: state.bears + 1 })),
  decrease: () => set((state) => ({ bears: state.bears - 1 })),
  reset: () => set({ bears: 0 }),
}))

// Usage - component only re-renders when selected state changes
function BearCounter() {
  const bears = useBearStore((state) => state.bears)
  return <h1>{bears} bears</h1>
}
```

### Slices Pattern (Modular Stores)

Split large stores into composable slices:

```tsx
// fishSlice.ts
export const createFishSlice = (set) => ({
  fishes: 0,
  addFish: () => set((state) => ({ fishes: state.fishes + 1 })),
})

// bearSlice.ts  
export const createBearSlice = (set) => ({
  bears: 0,
  addBear: () => set((state) => ({ bears: state.bears + 1 })),
  eatFish: () => set((state) => ({ fishes: state.fishes - 1 })),
})

// store.ts - combine slices
import { create } from 'zustand'
import { createBearSlice } from './bearSlice'
import { createFishSlice } from './fishSlice'

export const useBoundStore = create((...a) => ({
  ...createBearSlice(...a),
  ...createFishSlice(...a),
}))
```

### Cross-Slice Actions

```tsx
export const createBearFishSlice = (set, get) => ({
  addBearAndFish: () => {
    get().addBear()
    get().addFish()
  },
})
```

## Middleware

### Persist Middleware

```tsx
import { create } from 'zustand'
import { persist, createJSONStorage } from 'zustand/middleware'

const useStore = create(
  persist(
    (set, get) => ({
      bears: 0,
      addABear: () => set({ bears: get().bears + 1 }),
    }),
    {
      name: 'bear-storage', // unique key in storage
      storage: createJSONStorage(() => localStorage), // default
      partialize: (state) => ({ bears: state.bears }), // only persist specific fields
      version: 1, // for migrations
      migrate: (persistedState, version) => {
        if (version === 0) {
          // handle migration from v0
          persistedState.newField = persistedState.oldField
          delete persistedState.oldField
        }
        return persistedState
      },
    },
  ),
)
```

### Immer Middleware

```tsx
import { create } from 'zustand'
import { immer } from 'zustand/middleware/immer'

const useStore = create(
  immer((set) => ({
    users: [],
    addUser: (user) =>
      set((state) => {
        state.users.push(user) // mutate directly with immer
      }),
  })),
)
```

### Devtools Middleware

```tsx
import { create } from 'zustand'
import { devtools } from 'zustand/middleware'

const useStore = create(
  devtools(
    (set) => ({
      bears: 0,
      increase: () => set((state) => ({ bears: state.bears + 1 }), false, 'increase'),
    }),
    { name: 'BearStore' },
  ),
)
```

### Combining Middleware

```tsx
import { create } from 'zustand'
import { devtools, persist } from 'zustand/middleware'
import { immer } from 'zustand/middleware/immer'

const useStore = create(
  devtools(
    persist(
      immer((set) => ({
        // state and actions
      })),
      { name: 'store' },
    ),
    { name: 'MyStore' },
  ),
)
```

## Advanced Patterns

### Auto-Generated Selectors

```tsx
import { StoreApi, UseBoundStore } from 'zustand'

type WithSelectors<S> = S extends { getState: () => infer T }
  ? S & { use: { [K in keyof T]: () => T[K] } }
  : never

const createSelectors = <S extends UseBoundStore<StoreApi<object>>>(store: S) => {
  const storeWithSelectors = store as WithSelectors<typeof store>
  storeWithSelectors.use = {}
  for (const k of Object.keys(store.getState())) {
    ;(storeWithSelectors.use as any)[k] = () => store((s) => s[k as keyof typeof s])
  }
  return storeWithSelectors
}

// Usage
const useBearStore = createSelectors(useBearStoreBase)
const bears = useBearStore.use.bears() // auto-generated selector
```

### Actions Outside Components

```tsx
// Define actions at module level
export const useBoundStore = create(() => ({
  count: 0,
  text: 'hello',
}))

export const inc = () =>
  useBoundStore.setState((state) => ({ count: state.count + 1 }))

export const setText = (text) => useBoundStore.setState({ text })

// Advantages:
// - No hook needed to call an action
// - Facilitates code splitting
```

### Subscribe to State Changes

```tsx
// Subscribe outside React
const unsub = useBearStore.subscribe(
  (state) => state.bears,
  (bears, prevBears) => console.log('Bears changed:', prevBears, '->', bears),
)

// Cleanup
unsub()
```

### Transient Updates (No Re-render)

```tsx
const useStore = create((set, get) => ({
  scratches: 0,
  // This won't cause re-renders
  scratch: () => {
    console.log('Scratching without re-render')
  },
}))

// Direct access without causing re-renders
const scratches = useStore.getState().scratches
```

## Next.js / SSR Considerations

Handle hydration mismatch with async storage:

```tsx
// useStore.ts
import { useState, useEffect } from 'react'

const useStore = <T, F>(
  store: (callback: (state: T) => unknown) => unknown,
  callback: (state: T) => F,
) => {
  const result = store(callback) as F
  const [data, setData] = useState<F>()

  useEffect(() => {
    setData(result)
  }, [result])

  return data
}

// Usage in component
const bears = useStore(useBearStore, (state) => state.bears)
```

### Check Hydration Status

```tsx
const useHydration = () => {
  const [hydrated, setHydrated] = useState(false)

  useEffect(() => {
    const unsubFinishHydration = useBoundStore.persist.onFinishHydration(() => 
      setHydrated(true)
    )
    setHydrated(useBoundStore.persist.hasHydrated())
    return () => unsubFinishHydration()
  }, [])

  return hydrated
}
```

## Best Practices

1. **Use selectors** - Only select what you need to prevent unnecessary re-renders
2. **Colocate actions with state** - Keep actions in the store for encapsulation
3. **Use slices for large stores** - Split by domain/feature
4. **Apply middleware in combined store** - Not in individual slices
5. **Use TypeScript** - Better inference and developer experience
6. **Avoid storing derived data** - Compute in selectors instead
