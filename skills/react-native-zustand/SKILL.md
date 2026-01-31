# React Native State Management with Zustand

Lightweight, fast state management for React Native apps.

## When to Apply

- Managing global app state
- Replacing Redux with simpler solution
- State persistence across sessions
- Shared state between screens

## Installation

```bash
npm install zustand

# For persistence
npm install zustand @react-native-async-storage/async-storage
```

## Basic Store

```tsx
import { create } from 'zustand';

interface CounterState {
  count: number;
  increment: () => void;
  decrement: () => void;
  reset: () => void;
}

const useCounterStore = create<CounterState>((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
  reset: () => set({ count: 0 }),
}));

// Usage in component
function Counter() {
  const count = useCounterStore((state) => state.count);
  const increment = useCounterStore((state) => state.increment);
  
  return (
    <View>
      <Text>{count}</Text>
      <Button title="+" onPress={increment} />
    </View>
  );
}
```

## TypeScript Patterns

### Typed Store
```tsx
interface User {
  id: string;
  name: string;
  email: string;
}

interface AuthState {
  user: User | null;
  token: string | null;
  isLoading: boolean;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
}

const useAuthStore = create<AuthState>((set, get) => ({
  user: null,
  token: null,
  isLoading: false,
  
  login: async (email, password) => {
    set({ isLoading: true });
    try {
      const response = await api.login(email, password);
      set({ 
        user: response.user, 
        token: response.token,
        isLoading: false,
      });
    } catch (error) {
      set({ isLoading: false });
      throw error;
    }
  },
  
  logout: () => {
    set({ user: null, token: null });
    // Clear secure storage, etc.
  },
}));
```

## Selectors

### Prevent Unnecessary Rerenders
```tsx
// ❌ Bad - rerenders on any state change
const state = useStore();

// ✅ Good - rerenders only when count changes
const count = useStore((state) => state.count);

// ✅ Multiple values with shallow compare
import { shallow } from 'zustand/shallow';

const { count, total } = useStore(
  (state) => ({ count: state.count, total: state.total }),
  shallow
);
```

### Derived State
```tsx
const useStore = create((set, get) => ({
  items: [],
  addItem: (item) => set((state) => ({ items: [...state.items, item] })),
  
  // Computed getter
  get totalPrice() {
    return get().items.reduce((sum, item) => sum + item.price, 0);
  },
}));
```

## Persistence

### With AsyncStorage
```tsx
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';

interface SettingsState {
  theme: 'light' | 'dark';
  notifications: boolean;
  setTheme: (theme: 'light' | 'dark') => void;
  setNotifications: (enabled: boolean) => void;
}

const useSettingsStore = create<SettingsState>()(
  persist(
    (set) => ({
      theme: 'light',
      notifications: true,
      setTheme: (theme) => set({ theme }),
      setNotifications: (notifications) => set({ notifications }),
    }),
    {
      name: 'settings-storage',
      storage: createJSONStorage(() => AsyncStorage),
    }
  )
);
```

### Partial Persistence
```tsx
const useStore = create(
  persist(
    (set) => ({
      user: null,
      token: null,
      tempData: null, // Won't be persisted
    }),
    {
      name: 'auth-storage',
      partialize: (state) => ({ 
        user: state.user, 
        token: state.token,
        // tempData excluded
      }),
    }
  )
);
```

### Hydration Handling
```tsx
const useStore = create(
  persist(
    (set) => ({
      // ...state
    }),
    {
      name: 'app-storage',
      onRehydrateStorage: () => (state, error) => {
        if (error) {
          console.log('Failed to hydrate:', error);
        } else {
          console.log('Hydration complete');
        }
      },
    }
  )
);

// Wait for hydration in component
function App() {
  const hasHydrated = useStore.persist.hasHydrated();
  
  if (!hasHydrated) {
    return <LoadingScreen />;
  }
  
  return <MainApp />;
}
```

## Middleware

### Immer for Immutable Updates
```tsx
import { create } from 'zustand';
import { immer } from 'zustand/middleware/immer';

const useStore = create(
  immer((set) => ({
    users: [],
    
    addUser: (user) => set((state) => {
      state.users.push(user); // Direct mutation OK with immer
    }),
    
    updateUser: (id, updates) => set((state) => {
      const user = state.users.find(u => u.id === id);
      if (user) {
        Object.assign(user, updates);
      }
    }),
  }))
);
```

### DevTools
```tsx
import { devtools } from 'zustand/middleware';

const useStore = create(
  devtools(
    (set) => ({
      // ...state
    }),
    { name: 'MyStore' }
  )
);
```

### Combining Middleware
```tsx
const useStore = create(
  devtools(
    persist(
      immer((set) => ({
        // ...state
      })),
      { name: 'storage-key' }
    ),
    { name: 'DevTools' }
  )
);
```

## Actions Outside Components

```tsx
// Access store outside React
const { increment, decrement } = useCounterStore.getState();

// Subscribe to changes
const unsubscribe = useCounterStore.subscribe(
  (state) => state.count,
  (count) => console.log('Count changed:', count)
);

// Set state directly
useCounterStore.setState({ count: 10 });
```

## Store Slices

### Split Large Stores
```tsx
// slices/userSlice.ts
export interface UserSlice {
  user: User | null;
  setUser: (user: User) => void;
}

export const createUserSlice = (set) => ({
  user: null,
  setUser: (user) => set({ user }),
});

// slices/cartSlice.ts
export interface CartSlice {
  items: CartItem[];
  addItem: (item: CartItem) => void;
}

export const createCartSlice = (set) => ({
  items: [],
  addItem: (item) => set((state) => ({ 
    items: [...state.items, item] 
  })),
});

// store.ts
import { createUserSlice, UserSlice } from './slices/userSlice';
import { createCartSlice, CartSlice } from './slices/cartSlice';

type StoreState = UserSlice & CartSlice;

const useStore = create<StoreState>()((...args) => ({
  ...createUserSlice(...args),
  ...createCartSlice(...args),
}));
```

## Common Patterns

### Async Actions
```tsx
const useProductStore = create((set, get) => ({
  products: [],
  isLoading: false,
  error: null,
  
  fetchProducts: async () => {
    set({ isLoading: true, error: null });
    try {
      const products = await api.getProducts();
      set({ products, isLoading: false });
    } catch (error) {
      set({ error: error.message, isLoading: false });
    }
  },
  
  refreshProducts: async () => {
    // Don't show loading for refresh
    try {
      const products = await api.getProducts();
      set({ products });
    } catch (error) {
      set({ error: error.message });
    }
  },
}));
```

### Optimistic Updates
```tsx
const useTodoStore = create((set, get) => ({
  todos: [],
  
  toggleTodo: async (id) => {
    const { todos } = get();
    const todo = todos.find(t => t.id === id);
    
    // Optimistic update
    set({
      todos: todos.map(t => 
        t.id === id ? { ...t, completed: !t.completed } : t
      ),
    });
    
    try {
      await api.toggleTodo(id);
    } catch (error) {
      // Revert on error
      set({
        todos: todos.map(t =>
          t.id === id ? { ...t, completed: todo.completed } : t
        ),
      });
    }
  },
}));
```

## Best Practices

1. **Keep stores focused** - One domain per store
2. **Use selectors** - Prevent unnecessary rerenders
3. **Type your stores** - Full TypeScript support
4. **Persist carefully** - Only persist what's needed
5. **Handle hydration** - Show loading during rehydration
6. **Use immer for complex state** - Simplifies nested updates
7. **Split large stores** - Use slices for organization
