# Zod Validation Context

Comprehensive guide to Zod schema validation, transformations, and patterns.

## Installation

```bash
npm install zod
```

## Basic Schemas

### Primitives
```typescript
import { z } from 'zod'

// Primitives
const stringSchema = z.string()
const numberSchema = z.number()
const booleanSchema = z.boolean()
const dateSchema = z.date()
const bigintSchema = z.bigint()
const symbolSchema = z.symbol()
const undefinedSchema = z.undefined()
const nullSchema = z.null()

// Literals
const literalSchema = z.literal('hello')
const enumSchema = z.enum(['pending', 'active', 'completed'])

// Any/Unknown
const anySchema = z.any()
const unknownSchema = z.unknown()
```

### String Validations
```typescript
z.string().min(1, 'Required')
z.string().max(100, 'Too long')
z.string().length(10, 'Must be exactly 10 characters')
z.string().email('Invalid email')
z.string().url('Invalid URL')
z.string().uuid('Invalid UUID')
z.string().cuid()
z.string().cuid2()
z.string().ulid()
z.string().regex(/^[a-z]+$/, 'Lowercase letters only')
z.string().includes('hello')
z.string().startsWith('https://')
z.string().endsWith('.com')
z.string().datetime()
z.string().ip()
z.string().trim()
z.string().toLowerCase()
z.string().toUpperCase()
```

### Number Validations
```typescript
z.number().int('Must be integer')
z.number().positive('Must be positive')
z.number().negative('Must be negative')
z.number().nonnegative('Must be >= 0')
z.number().nonpositive('Must be <= 0')
z.number().finite('Must be finite')
z.number().safe('Must be safe integer')
z.number().min(0, 'Min is 0')
z.number().max(100, 'Max is 100')
z.number().gt(0, 'Must be > 0')
z.number().gte(0, 'Must be >= 0')
z.number().lt(100, 'Must be < 100')
z.number().lte(100, 'Must be <= 100')
z.number().multipleOf(5, 'Must be multiple of 5')
```

## Object Schemas

### Basic Object
```typescript
const userSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1),
  email: z.string().email(),
  age: z.number().int().positive().optional(),
  role: z.enum(['user', 'admin', 'moderator']).default('user'),
  createdAt: z.date().default(() => new Date()),
})

// Infer TypeScript type
type User = z.infer<typeof userSchema>

// Parse (throws on error)
const user = userSchema.parse(data)

// Safe parse (returns result object)
const result = userSchema.safeParse(data)
if (result.success) {
  console.log(result.data) // User
} else {
  console.log(result.error.issues) // ZodIssue[]
}
```

### Object Modifiers
```typescript
// Make all fields optional
const partialUser = userSchema.partial()

// Make all fields required
const requiredUser = userSchema.required()

// Pick specific fields
const pickedUser = userSchema.pick({ name: true, email: true })

// Omit specific fields
const omittedUser = userSchema.omit({ password: true })

// Extend object
const extendedUser = userSchema.extend({
  phone: z.string().optional(),
})

// Merge objects
const mergedSchema = userSchema.merge(additionalSchema)

// Strict (no unknown keys)
const strictUser = userSchema.strict()

// Passthrough (keep unknown keys)
const passthroughUser = userSchema.passthrough()

// Strip (remove unknown keys)
const strippedUser = userSchema.strip()
```

## Arrays and Collections

```typescript
// Array of strings
const tagsSchema = z.array(z.string())

// With constraints
const constrainedArray = z.array(z.number())
  .min(1, 'At least one item')
  .max(10, 'Max 10 items')
  .nonempty('Cannot be empty')

// Tuple
const pointSchema = z.tuple([z.number(), z.number()])

// Set
const setSchema = z.set(z.string())

// Map
const mapSchema = z.map(z.string(), z.number())

// Record (string keys, typed values)
const recordSchema = z.record(z.string(), z.number())
```

## Union and Discriminated Union

```typescript
// Union
const stringOrNumber = z.union([z.string(), z.number()])
// Or using .or()
const stringOrNumber2 = z.string().or(z.number())

// Discriminated union (much better performance)
const resultSchema = z.discriminatedUnion('status', [
  z.object({
    status: z.literal('success'),
    data: z.object({ id: z.string() }),
  }),
  z.object({
    status: z.literal('error'),
    error: z.string(),
  }),
])

type Result = z.infer<typeof resultSchema>
// { status: 'success'; data: { id: string } } | { status: 'error'; error: string }
```

## Transformations

```typescript
// Transform after parsing
const numberFromString = z.string().transform((val) => parseInt(val, 10))

// Coercion (automatically convert)
z.coerce.string()   // Converts to string
z.coerce.number()   // Converts to number
z.coerce.boolean()  // Converts to boolean
z.coerce.date()     // Converts to Date

// Complex transformation
const userInputSchema = z.object({
  name: z.string().trim(),
  email: z.string().toLowerCase().email(),
  tags: z.string().transform((s) => s.split(',').map((t) => t.trim())),
})

// Preprocess (transform before validation)
const preprocessed = z.preprocess(
  (val) => (typeof val === 'string' ? JSON.parse(val) : val),
  z.object({ name: z.string() })
)
```

## Refinements and Custom Validation

```typescript
// Simple refinement
const passwordSchema = z.string()
  .min(8, 'At least 8 characters')
  .refine(
    (val) => /[A-Z]/.test(val),
    'Must contain uppercase letter'
  )
  .refine(
    (val) => /[0-9]/.test(val),
    'Must contain number'
  )

// Super refine (multiple errors)
const formSchema = z.object({
  password: z.string(),
  confirmPassword: z.string(),
}).superRefine((data, ctx) => {
  if (data.password !== data.confirmPassword) {
    ctx.addIssue({
      code: z.ZodIssueCode.custom,
      message: 'Passwords do not match',
      path: ['confirmPassword'],
    })
  }
})

// Async refinement
const usernameSchema = z.string().refine(
  async (username) => {
    const exists = await checkUsernameExists(username)
    return !exists
  },
  'Username already taken'
)
```

## Error Handling

```typescript
// Custom error messages
const schema = z.object({
  email: z.string({
    required_error: 'Email is required',
    invalid_type_error: 'Email must be a string',
  }).email({ message: 'Invalid email format' }),
})

// Format errors
import { fromZodError } from 'zod-validation-error'

const result = schema.safeParse(data)
if (!result.success) {
  const formatted = fromZodError(result.error)
  console.log(formatted.message)
}

// Custom error map
const customErrorMap: z.ZodErrorMap = (issue, ctx) => {
  if (issue.code === z.ZodIssueCode.invalid_type) {
    return { message: `Expected ${issue.expected}, got ${issue.received}` }
  }
  return { message: ctx.defaultError }
}

z.setErrorMap(customErrorMap)
```

## Common Patterns

### API Request/Response
```typescript
const createUserRequestSchema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
  password: z.string().min(8),
})

const createUserResponseSchema = z.object({
  id: z.string().uuid(),
  name: z.string(),
  email: z.string(),
  createdAt: z.string().datetime(),
})

type CreateUserRequest = z.infer<typeof createUserRequestSchema>
type CreateUserResponse = z.infer<typeof createUserResponseSchema>

// Validate request
function createUser(input: unknown): CreateUserResponse {
  const data = createUserRequestSchema.parse(input)
  // ... create user
  return createUserResponseSchema.parse(response)
}
```

### Form Validation
```typescript
const loginFormSchema = z.object({
  email: z.string().email('Please enter a valid email'),
  password: z.string().min(1, 'Password is required'),
  rememberMe: z.boolean().default(false),
})

// With React Hook Form
import { zodResolver } from '@hookform/resolvers/zod'
import { useForm } from 'react-hook-form'

const form = useForm({
  resolver: zodResolver(loginFormSchema),
  defaultValues: {
    email: '',
    password: '',
    rememberMe: false,
  },
})
```

### Configuration Validation
```typescript
const configSchema = z.object({
  port: z.coerce.number().int().min(1).max(65535).default(3000),
  host: z.string().default('localhost'),
  database: z.object({
    url: z.string().url(),
    pool: z.coerce.number().int().positive().default(10),
  }),
  features: z.object({
    darkMode: z.coerce.boolean().default(false),
    analytics: z.coerce.boolean().default(true),
  }),
})

const config = configSchema.parse({
  port: process.env.PORT,
  host: process.env.HOST,
  database: {
    url: process.env.DATABASE_URL,
    pool: process.env.DB_POOL,
  },
  features: {
    darkMode: process.env.DARK_MODE,
    analytics: process.env.ANALYTICS,
  },
})
```

## Best Practices

1. **Infer types** - Use `z.infer<typeof schema>` instead of manual types
2. **Use discriminated unions** - Better performance than regular unions
3. **Coerce when needed** - Use `z.coerce` for form inputs
4. **Custom error messages** - Always provide user-friendly messages
5. **Reuse schemas** - Create base schemas and extend them
6. **Validate at boundaries** - API routes, form submissions, config
7. **Use safeParse** - Handle errors gracefully in user-facing code
8. **Transform early** - Clean data during validation, not after
