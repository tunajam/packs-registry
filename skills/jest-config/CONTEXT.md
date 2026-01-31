# Jest Configuration Context

Comprehensive guide to Jest configuration for TypeScript and React projects.

## Setup

```bash
npm install -D jest @types/jest ts-jest @jest/globals
# For React
npm install -D @testing-library/react @testing-library/jest-dom jest-environment-jsdom
```

## Configuration

### Basic TypeScript Config
```javascript
// jest.config.js
/** @type {import('jest').Config} */
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src'],
  testMatch: ['**/__tests__/**/*.ts', '**/*.{test,spec}.ts'],
  transform: {
    '^.+\\.tsx?$': ['ts-jest', {
      tsconfig: 'tsconfig.json',
    }],
  },
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
  },
  collectCoverageFrom: [
    'src/**/*.{ts,tsx}',
    '!src/**/*.d.ts',
    '!src/**/index.ts',
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
  clearMocks: true,
  resetMocks: true,
}
```

### React + TypeScript Config
```javascript
// jest.config.js
/** @type {import('jest').Config} */
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'jsdom',
  setupFilesAfterEnv: ['<rootDir>/src/test/setup.ts'],
  roots: ['<rootDir>/src'],
  testMatch: ['**/__tests__/**/*.{ts,tsx}', '**/*.{test,spec}.{ts,tsx}'],
  transform: {
    '^.+\\.tsx?$': ['ts-jest', {
      tsconfig: 'tsconfig.json',
      useESM: true,
    }],
  },
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
    '\\.(css|less|scss|sass)$': 'identity-obj-proxy',
    '\\.(jpg|jpeg|png|gif|svg)$': '<rootDir>/src/test/__mocks__/fileMock.js',
  },
  collectCoverageFrom: [
    'src/**/*.{ts,tsx}',
    '!src/**/*.d.ts',
    '!src/**/index.ts',
    '!src/test/**/*',
  ],
  testPathIgnorePatterns: ['/node_modules/', '/dist/'],
  moduleFileExtensions: ['ts', 'tsx', 'js', 'jsx', 'json'],
}
```

### ESM Configuration
```javascript
// jest.config.mjs
export default {
  preset: 'ts-jest/presets/default-esm',
  testEnvironment: 'node',
  extensionsToTreatAsEsm: ['.ts'],
  moduleNameMapper: {
    '^(\\.{1,2}/.*)\\.js$': '$1',
  },
  transform: {
    '^.+\\.tsx?$': ['ts-jest', {
      useESM: true,
    }],
  },
}
```

## Test Setup

```typescript
// src/test/setup.ts
import '@testing-library/jest-dom'

// Mock window.matchMedia
Object.defineProperty(window, 'matchMedia', {
  writable: true,
  value: jest.fn().mockImplementation(query => ({
    matches: false,
    media: query,
    onchange: null,
    addListener: jest.fn(),
    removeListener: jest.fn(),
    addEventListener: jest.fn(),
    removeEventListener: jest.fn(),
    dispatchEvent: jest.fn(),
  })),
})

// Mock IntersectionObserver
global.IntersectionObserver = jest.fn().mockImplementation(() => ({
  observe: jest.fn(),
  unobserve: jest.fn(),
  disconnect: jest.fn(),
}))

// Mock ResizeObserver
global.ResizeObserver = jest.fn().mockImplementation(() => ({
  observe: jest.fn(),
  unobserve: jest.fn(),
  disconnect: jest.fn(),
}))

// Reset mocks after each test
afterEach(() => {
  jest.clearAllMocks()
})
```

```javascript
// src/test/__mocks__/fileMock.js
module.exports = 'test-file-stub'
```

## Test Patterns

### Basic Tests
```typescript
import { describe, it, expect, beforeEach, afterEach, jest } from '@jest/globals'

describe('Calculator', () => {
  let calculator: Calculator

  beforeEach(() => {
    calculator = new Calculator()
  })

  afterEach(() => {
    jest.clearAllMocks()
  })

  it('should add two numbers', () => {
    expect(calculator.add(2, 3)).toBe(5)
  })

  it('should throw on division by zero', () => {
    expect(() => calculator.divide(1, 0)).toThrow('Division by zero')
  })
})
```

### Async Tests
```typescript
describe('async operations', () => {
  it('should resolve promise', async () => {
    const result = await fetchData()
    expect(result).toBeDefined()
  })

  it('should reject with error', async () => {
    await expect(fetchBadData()).rejects.toThrow('Not found')
  })

  it('should handle callback', (done) => {
    fetchWithCallback((data) => {
      expect(data).toBeDefined()
      done()
    })
  })
})
```

### Mocking
```typescript
// Mock module
jest.mock('@/lib/api', () => ({
  fetchUser: jest.fn(),
  fetchPosts: jest.fn(),
}))

import { fetchUser, fetchPosts } from '@/lib/api'

const mockFetchUser = fetchUser as jest.MockedFunction<typeof fetchUser>

describe('UserService', () => {
  beforeEach(() => {
    jest.clearAllMocks()
  })

  it('should fetch user', async () => {
    const mockUser = { id: '1', name: 'John' }
    mockFetchUser.mockResolvedValue(mockUser)

    const user = await userService.getUser('1')

    expect(fetchUser).toHaveBeenCalledWith('1')
    expect(user).toEqual(mockUser)
  })
})

// Spy on method
it('should call internal method', () => {
  const spy = jest.spyOn(service, 'validate')
  
  service.process(data)
  
  expect(spy).toHaveBeenCalledWith(data)
  spy.mockRestore()
})

// Mock implementation
it('should use mock implementation', () => {
  const mockFn = jest.fn()
    .mockReturnValueOnce(1)
    .mockReturnValueOnce(2)
    .mockReturnValue(3)

  expect(mockFn()).toBe(1)
  expect(mockFn()).toBe(2)
  expect(mockFn()).toBe(3)
})
```

### Timer Mocks
```typescript
describe('timers', () => {
  beforeEach(() => {
    jest.useFakeTimers()
  })

  afterEach(() => {
    jest.useRealTimers()
  })

  it('should handle setTimeout', () => {
    const callback = jest.fn()
    setTimeout(callback, 1000)

    expect(callback).not.toHaveBeenCalled()
    
    jest.advanceTimersByTime(1000)
    
    expect(callback).toHaveBeenCalledTimes(1)
  })

  it('should run all timers', () => {
    const callback = jest.fn()
    setTimeout(callback, 5000)
    
    jest.runAllTimers()
    
    expect(callback).toHaveBeenCalled()
  })
})
```

### React Component Testing
```typescript
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

  it('should call onChange', async () => {
    const onChange = jest.fn()
    const user = userEvent.setup()
    render(<Counter initialCount={0} onChange={onChange} />)
    
    await user.click(screen.getByRole('button', { name: /increment/i }))
    
    expect(onChange).toHaveBeenCalledWith(1)
  })
})
```

### Snapshot Testing
```typescript
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
      `"<button class=\\"btn btn-primary\\">Primary</button>"`
    )
  })
})
```

### Testing Hooks
```typescript
import { renderHook, act } from '@testing-library/react'
import { useCounter } from './useCounter'

describe('useCounter', () => {
  it('should initialize with value', () => {
    const { result } = renderHook(() => useCounter(10))
    expect(result.current.count).toBe(10)
  })

  it('should increment', () => {
    const { result } = renderHook(() => useCounter(0))
    
    act(() => {
      result.current.increment()
    })
    
    expect(result.current.count).toBe(1)
  })
})
```

## CLI Commands

```bash
# Run tests
npx jest

# Watch mode
npx jest --watch

# Coverage
npx jest --coverage

# Run specific file
npx jest src/utils/math.test.ts

# Run matching pattern
npx jest -t "should add"

# Update snapshots
npx jest -u

# Verbose output
npx jest --verbose

# Run in band (sequential)
npx jest --runInBand
```

## Package.json Scripts

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:ci": "jest --ci --coverage --maxWorkers=2"
  }
}
```

## Best Practices

1. **Isolate tests** - Each test should be independent
2. **Use descriptive names** - Test names should explain behavior
3. **Mock external dependencies** - Database, APIs, etc.
4. **Clean up after tests** - Reset mocks, clear timers
5. **Test behavior, not implementation** - Focus on outputs
6. **Use testing-library** - Test user interactions
7. **Set coverage thresholds** - Maintain code quality
8. **Keep tests fast** - Mock slow operations
