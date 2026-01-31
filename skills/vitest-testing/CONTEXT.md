# Vitest Testing Context

Comprehensive guide to Vitest configuration and testing patterns.

## Setup

```bash
npm install -D vitest @vitest/coverage-v8 @vitest/ui
```

## Configuration

### Basic Configuration
```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import tsconfigPaths from 'vite-tsconfig-paths'

export default defineConfig({
  plugins: [react(), tsconfigPaths()],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: ['./src/test/setup.ts'],
    include: ['src/**/*.{test,spec}.{js,ts,jsx,tsx}'],
    exclude: ['node_modules', 'dist', '.idea', '.git', '.cache'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      exclude: [
        'node_modules/',
        'src/test/',
        '**/*.d.ts',
        '**/*.config.*',
        '**/types/*',
      ],
    },
  },
})
```

### Workspace Configuration
```typescript
// vitest.workspace.ts
import { defineWorkspace } from 'vitest/config'

export default defineWorkspace([
  {
    extends: './vitest.config.ts',
    test: {
      name: 'unit',
      include: ['src/**/*.{test,spec}.ts'],
    },
  },
  {
    extends: './vitest.config.ts',
    test: {
      name: 'integration',
      include: ['tests/integration/**/*.{test,spec}.ts'],
      environment: 'node',
    },
  },
])
```

## Test Setup

```typescript
// src/test/setup.ts
import '@testing-library/jest-dom/vitest'
import { cleanup } from '@testing-library/react'
import { afterEach, vi } from 'vitest'

// Cleanup after each test
afterEach(() => {
  cleanup()
})

// Mock window.matchMedia
Object.defineProperty(window, 'matchMedia', {
  writable: true,
  value: vi.fn().mockImplementation(query => ({
    matches: false,
    media: query,
    onchange: null,
    addListener: vi.fn(),
    removeListener: vi.fn(),
    addEventListener: vi.fn(),
    removeEventListener: vi.fn(),
    dispatchEvent: vi.fn(),
  })),
})

// Mock ResizeObserver
global.ResizeObserver = vi.fn().mockImplementation(() => ({
  observe: vi.fn(),
  unobserve: vi.fn(),
  disconnect: vi.fn(),
}))

// Mock IntersectionObserver
global.IntersectionObserver = vi.fn().mockImplementation(() => ({
  observe: vi.fn(),
  unobserve: vi.fn(),
  disconnect: vi.fn(),
}))
```

## Test Patterns

### Basic Test Structure
```typescript
import { describe, it, expect, beforeEach, afterEach, vi } from 'vitest'

describe('Calculator', () => {
  let calculator: Calculator

  beforeEach(() => {
    calculator = new Calculator()
  })

  afterEach(() => {
    vi.clearAllMocks()
  })

  describe('add', () => {
    it('should add two positive numbers', () => {
      expect(calculator.add(2, 3)).toBe(5)
    })

    it('should handle negative numbers', () => {
      expect(calculator.add(-1, 1)).toBe(0)
    })
  })

  describe('divide', () => {
    it('should throw on division by zero', () => {
      expect(() => calculator.divide(1, 0)).toThrow('Division by zero')
    })
  })
})
```

### Async Tests
```typescript
import { describe, it, expect, vi } from 'vitest'

describe('async operations', () => {
  it('should resolve promise', async () => {
    const result = await fetchData()
    expect(result).toBeDefined()
  })

  it('should reject with error', async () => {
    await expect(fetchBadData()).rejects.toThrow('Not found')
  })

  it('should handle callback', () => {
    return new Promise<void>((resolve) => {
      fetchWithCallback((data) => {
        expect(data).toBeDefined()
        resolve()
      })
    })
  })
})
```

### Mocking

```typescript
import { describe, it, expect, vi, beforeEach, type Mock } from 'vitest'

// Mock module
vi.mock('@/lib/api', () => ({
  fetchUser: vi.fn(),
  fetchPosts: vi.fn(),
}))

import { fetchUser, fetchPosts } from '@/lib/api'

describe('UserService', () => {
  beforeEach(() => {
    vi.clearAllMocks()
  })

  it('should fetch user', async () => {
    const mockUser = { id: '1', name: 'John' }
    ;(fetchUser as Mock).mockResolvedValue(mockUser)

    const user = await userService.getUser('1')

    expect(fetchUser).toHaveBeenCalledWith('1')
    expect(user).toEqual(mockUser)
  })

  it('should handle error', async () => {
    ;(fetchUser as Mock).mockRejectedValue(new Error('Network error'))

    await expect(userService.getUser('1')).rejects.toThrow('Network error')
  })
})

// Spy on method
it('should call internal method', () => {
  const spy = vi.spyOn(service, 'validate')
  
  service.process(data)
  
  expect(spy).toHaveBeenCalledWith(data)
  spy.mockRestore()
})

// Mock implementation
it('should use mock implementation', () => {
  const mockFn = vi.fn()
    .mockReturnValueOnce(1)
    .mockReturnValueOnce(2)
    .mockReturnValue(3)

  expect(mockFn()).toBe(1)
  expect(mockFn()).toBe(2)
  expect(mockFn()).toBe(3)
  expect(mockFn()).toBe(3)
})
```

### Timer Mocks
```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest'

describe('timers', () => {
  beforeEach(() => {
    vi.useFakeTimers()
  })

  afterEach(() => {
    vi.useRealTimers()
  })

  it('should handle setTimeout', () => {
    const callback = vi.fn()
    setTimeout(callback, 1000)

    expect(callback).not.toHaveBeenCalled()
    
    vi.advanceTimersByTime(1000)
    
    expect(callback).toHaveBeenCalledOnce()
  })

  it('should handle setInterval', () => {
    const callback = vi.fn()
    setInterval(callback, 1000)

    vi.advanceTimersByTime(3000)
    
    expect(callback).toHaveBeenCalledTimes(3)
  })

  it('should run all timers', async () => {
    const promise = new Promise(resolve => setTimeout(resolve, 5000))
    
    vi.runAllTimers()
    
    await expect(promise).resolves.toBeUndefined()
  })
})
```

### React Component Testing
```typescript
import { describe, it, expect, vi } from 'vitest'
import { render, screen, fireEvent, waitFor } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { Counter } from './Counter'

describe('Counter', () => {
  it('should render initial count', () => {
    render(<Counter initialCount={5} />)
    
    expect(screen.getByText('Count: 5')).toBeInTheDocument()
  })

  it('should increment on click', async () => {
    const user = userEvent.setup()
    render(<Counter initialCount={0} />)
    
    await user.click(screen.getByRole('button', { name: /increment/i }))
    
    expect(screen.getByText('Count: 1')).toBeInTheDocument()
  })

  it('should call onChange when count changes', async () => {
    const onChange = vi.fn()
    const user = userEvent.setup()
    render(<Counter initialCount={0} onChange={onChange} />)
    
    await user.click(screen.getByRole('button', { name: /increment/i }))
    
    expect(onChange).toHaveBeenCalledWith(1)
  })
})
```

### Snapshot Testing
```typescript
import { describe, it, expect } from 'vitest'
import { render } from '@testing-library/react'
import { Button } from './Button'

describe('Button', () => {
  it('should match snapshot', () => {
    const { container } = render(<Button>Click me</Button>)
    expect(container).toMatchSnapshot()
  })

  it('should match inline snapshot', () => {
    const { container } = render(<Button variant="primary">Primary</Button>)
    expect(container.innerHTML).toMatchInlineSnapshot(
      `"<button class="btn btn-primary">Primary</button>"`
    )
  })
})
```

### Testing Hooks
```typescript
import { describe, it, expect } from 'vitest'
import { renderHook, act } from '@testing-library/react'
import { useCounter } from './useCounter'

describe('useCounter', () => {
  it('should initialize with default value', () => {
    const { result } = renderHook(() => useCounter())
    
    expect(result.current.count).toBe(0)
  })

  it('should initialize with custom value', () => {
    const { result } = renderHook(() => useCounter(10))
    
    expect(result.current.count).toBe(10)
  })

  it('should increment', () => {
    const { result } = renderHook(() => useCounter())
    
    act(() => {
      result.current.increment()
    })
    
    expect(result.current.count).toBe(1)
  })
})
```

### Test Utilities
```typescript
// test/utils.tsx
import { render, type RenderOptions } from '@testing-library/react'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { ReactElement } from 'react'

const createTestQueryClient = () => new QueryClient({
  defaultOptions: {
    queries: { retry: false },
    mutations: { retry: false },
  },
})

function AllTheProviders({ children }: { children: React.ReactNode }) {
  const queryClient = createTestQueryClient()
  return (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  )
}

const customRender = (
  ui: ReactElement,
  options?: Omit<RenderOptions, 'wrapper'>
) => render(ui, { wrapper: AllTheProviders, ...options })

export * from '@testing-library/react'
export { customRender as render }
```

## CLI Commands

```bash
# Run tests
npx vitest

# Run once (CI)
npx vitest run

# Watch mode
npx vitest watch

# UI mode
npx vitest --ui

# Coverage
npx vitest --coverage

# Run specific file
npx vitest src/utils/math.test.ts

# Run tests matching pattern
npx vitest -t "should add"

# Update snapshots
npx vitest -u
```

## Best Practices

1. **Test behavior, not implementation** - Focus on what, not how
2. **Use descriptive test names** - Should read like sentences
3. **One assertion per test (mostly)** - Keep tests focused
4. **Mock external dependencies** - Database, APIs, etc.
5. **Use testing-library queries** - getByRole > getByTestId
6. **Avoid testing implementation details** - Test public API
7. **Clean up after tests** - Reset mocks, cleanup DOM
8. **Use test utilities** - Create custom render with providers
