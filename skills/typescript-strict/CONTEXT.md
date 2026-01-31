# TypeScript Strict Configuration Context

Comprehensive guide to strict TypeScript configuration and type-safe patterns.

## Recommended tsconfig.json (Strict)

```json
{
  "compilerOptions": {
    // Strict Type Checking
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "useUnknownInCatchVariables": true,
    "alwaysStrict": true,
    
    // Additional Strict Checks
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    
    // Module Resolution
    "module": "ESNext",
    "moduleResolution": "bundler",
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    
    // Output
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "skipLibCheck": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    
    // Path Aliases
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

## Strict Flags Explained

### `strict: true`
Enables all strict type-checking options:
- `noImplicitAny`
- `strictNullChecks`
- `strictFunctionTypes`
- `strictBindCallApply`
- `strictPropertyInitialization`
- `noImplicitThis`
- `alwaysStrict`

### `noUncheckedIndexedAccess`
Adds `undefined` to index signature access:

```typescript
const arr: string[] = []
const item = arr[0] // string | undefined (with flag)
                    // string (without flag - dangerous!)

const obj: Record<string, number> = {}
const value = obj["key"] // number | undefined (with flag)
```

### `exactOptionalPropertyTypes`
Distinguishes between `undefined` and missing:

```typescript
interface User {
  name: string
  nickname?: string // Can be missing, but NOT undefined
}

const user: User = { name: "John" } // ✅
const user2: User = { name: "John", nickname: undefined } // ❌ Error!
```

### `noPropertyAccessFromIndexSignature`
Forces bracket notation for index signatures:

```typescript
interface Config {
  known: string
  [key: string]: string
}

const config: Config = { known: "value" }
config.known      // ✅ OK - declared property
config["unknown"] // ✅ OK - bracket notation
config.unknown    // ❌ Error - must use brackets
```

## Type Utility Patterns

### Make Properties Required
```typescript
type RequiredProps<T, K extends keyof T> = T & Required<Pick<T, K>>

interface User {
  id?: string
  name?: string
  email?: string
}

type UserWithId = RequiredProps<User, "id">
// { id: string; name?: string; email?: string }
```

### Deep Partial
```typescript
type DeepPartial<T> = T extends object
  ? { [P in keyof T]?: DeepPartial<T[P]> }
  : T

interface Config {
  server: {
    port: number
    host: string
  }
}

type PartialConfig = DeepPartial<Config>
// { server?: { port?: number; host?: string } }
```

### Non-Nullable Object
```typescript
type NonNullableObject<T> = {
  [K in keyof T]: NonNullable<T[K]>
}
```

### Strict Omit
```typescript
type StrictOmit<T, K extends keyof T> = Omit<T, K>
// Unlike Omit, this errors if K isn't in T
```

### Extract Function Return Type
```typescript
type AsyncReturnType<T extends (...args: any) => Promise<any>> =
  T extends (...args: any) => Promise<infer R> ? R : never
```

## Type Guard Patterns

### `is` Type Guards
```typescript
function isString(value: unknown): value is string {
  return typeof value === "string"
}

function isUser(value: unknown): value is User {
  return (
    typeof value === "object" &&
    value !== null &&
    "id" in value &&
    "name" in value
  )
}
```

### `asserts` Type Guards
```typescript
function assertIsString(value: unknown): asserts value is string {
  if (typeof value !== "string") {
    throw new Error("Expected string")
  }
}

function assertDefined<T>(value: T | null | undefined): asserts value is T {
  if (value === null || value === undefined) {
    throw new Error("Value is not defined")
  }
}
```

### Exhaustive Checks
```typescript
function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${value}`)
}

type Status = "pending" | "success" | "error"

function handleStatus(status: Status) {
  switch (status) {
    case "pending":
      return "Loading..."
    case "success":
      return "Done!"
    case "error":
      return "Failed"
    default:
      return assertNever(status) // Compile error if case missing
  }
}
```

## Branded Types

```typescript
type Brand<T, B> = T & { __brand: B }

type UserId = Brand<string, "UserId">
type PostId = Brand<string, "PostId">

function createUserId(id: string): UserId {
  return id as UserId
}

function getUser(id: UserId) { /* ... */ }
function getPost(id: PostId) { /* ... */ }

const userId = createUserId("123")
getUser(userId) // ✅
getPost(userId) // ❌ Type error!
```

## Const Assertions

```typescript
// Without as const
const routes = {
  home: "/",
  about: "/about",
}
// type: { home: string; about: string }

// With as const
const routes = {
  home: "/",
  about: "/about",
} as const
// type: { readonly home: "/"; readonly about: "/about" }

// Array as const
const statuses = ["pending", "active", "done"] as const
type Status = (typeof statuses)[number] // "pending" | "active" | "done"
```

## Template Literal Types

```typescript
type HTTPMethod = "GET" | "POST" | "PUT" | "DELETE"
type Endpoint = "/users" | "/posts"

type Route = `${HTTPMethod} ${Endpoint}`
// "GET /users" | "GET /posts" | "POST /users" | ...

type EventName<T extends string> = `on${Capitalize<T>}`
type ClickEvent = EventName<"click"> // "onClick"
```

## Discriminated Unions

```typescript
type Result<T, E = Error> =
  | { success: true; data: T }
  | { success: false; error: E }

function handleResult<T>(result: Result<T>) {
  if (result.success) {
    console.log(result.data) // T - TypeScript knows
  } else {
    console.error(result.error) // E - TypeScript knows
  }
}

// API Response Pattern
type ApiResponse<T> =
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: string }
```

## Safe Record Access

```typescript
// Problem: Record allows any key
const users: Record<string, User> = {}
users["nonexistent"].name // No error, but crashes!

// Solution 1: noUncheckedIndexedAccess
// Solution 2: Map
const usersMap = new Map<string, User>()
usersMap.get("nonexistent")?.name // Safe - returns undefined

// Solution 3: Explicit undefined
const users: Record<string, User | undefined> = {}
```

## Function Overloads

```typescript
function parse(input: string): string[]
function parse(input: string[]): string[][]
function parse(input: string | string[]): string[] | string[][] {
  if (typeof input === "string") {
    return input.split(",")
  }
  return input.map(s => s.split(","))
}
```

## Generic Constraints

```typescript
// Basic constraint
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key]
}

// Multiple constraints
function merge<T extends object, U extends object>(a: T, b: U): T & U {
  return { ...a, ...b }
}

// Conditional constraints
type ArrayOrSingle<T> = T extends any[] ? T : T[]
```

## Best Practices

1. **Start strict** - Enable all strict options from day one
2. **Avoid `any`** - Use `unknown` for truly unknown types
3. **Prefer `interface` for objects** - `type` for unions, functions
4. **Use const assertions** - For literal types and readonly arrays
5. **Enable `verbatimModuleSyntax`** - Explicit import/export types
6. **Don't disable strict checks** - Fix the underlying issues instead
7. **Use branded types** - For IDs and domain-specific strings
8. **Exhaustive switch** - Use `assertNever` for compile-time safety
