# Skill: React Hook Form + Zod

Build forms with React Hook Form for state management and Zod for validation.

## When to Use
- Building forms in React/Next.js
- Need client-side validation
- Want type-safe form data
- Complex forms with multiple fields

## Setup

```bash
npm install react-hook-form zod @hookform/resolvers
```

## Basic Pattern

```tsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

// 1. Define schema
const schema = z.object({
  email: z.string().email("Invalid email"),
  password: z.string().min(8, "Password must be 8+ characters"),
});

// 2. Infer type from schema
type FormData = z.infer<typeof schema>;

// 3. Use in component
function LoginForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<FormData>({
    resolver: zodResolver(schema),
  });

  const onSubmit = async (data: FormData) => {
    // data is fully typed!
    await login(data.email, data.password);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register("email")} placeholder="Email" />
      {errors.email && <span>{errors.email.message}</span>}

      <input {...register("password")} type="password" placeholder="Password" />
      {errors.password && <span>{errors.password.message}</span>}

      <button disabled={isSubmitting}>
        {isSubmitting ? "Loading..." : "Log in"}
      </button>
    </form>
  );
}
```

## Common Zod Patterns

### String Validations
```typescript
z.string().email()                    // Email format
z.string().url()                      // URL format
z.string().min(1, "Required")         // Non-empty (required field)
z.string().min(8).max(100)            // Length range
z.string().regex(/^[a-z]+$/)          // Regex pattern
z.string().trim()                     // Auto-trim whitespace
z.string().toLowerCase()              // Transform to lowercase
```

### Number Validations
```typescript
z.number().min(0)                     // Non-negative
z.number().max(100)                   // Maximum value
z.number().int()                      // Integer only
z.number().positive()                 // > 0
z.coerce.number()                     // Parse string to number
```

### Optional and Nullable
```typescript
z.string().optional()                 // string | undefined
z.string().nullable()                 // string | null
z.string().nullish()                  // string | null | undefined
z.string().default("hello")           // Default value
```

### Objects and Arrays
```typescript
z.object({
  name: z.string(),
  age: z.number(),
})

z.array(z.string())                   // string[]
z.array(z.string()).min(1)            // At least one item
z.array(z.string()).max(10)           // At most 10 items
```

### Enums and Unions
```typescript
z.enum(["admin", "user", "guest"])    // Literal union
z.union([z.string(), z.number()])     // string | number
z.literal("active")                   // Exact value
```

### Custom Validations
```typescript
z.string().refine(
  (val) => val.includes("@"),
  { message: "Must contain @" }
)

// Async validation
z.string().refine(
  async (email) => !(await checkEmailExists(email)),
  { message: "Email already registered" }
)
```

## Advanced Form Patterns

### With shadcn/ui Form Components
```tsx
import { Form, FormControl, FormField, FormItem, FormLabel, FormMessage } from "@/components/ui/form";
import { Input } from "@/components/ui/input";
import { Button } from "@/components/ui/button";

function SignupForm() {
  const form = useForm<FormData>({
    resolver: zodResolver(schema),
    defaultValues: { email: "", password: "" },
  });

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
        <FormField
          control={form.control}
          name="email"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Email</FormLabel>
              <FormControl>
                <Input placeholder="you@example.com" {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        <Button type="submit" disabled={form.formState.isSubmitting}>
          Submit
        </Button>
      </form>
    </Form>
  );
}
```

### Dynamic Fields (useFieldArray)
```tsx
const { fields, append, remove } = useFieldArray({
  control: form.control,
  name: "items",
});

return (
  <>
    {fields.map((field, index) => (
      <div key={field.id}>
        <input {...register(`items.${index}.name`)} />
        <button type="button" onClick={() => remove(index)}>Remove</button>
      </div>
    ))}
    <button type="button" onClick={() => append({ name: "" })}>Add Item</button>
  </>
);
```

### Watch Values
```tsx
const email = form.watch("email");  // Watch single field
const allValues = form.watch();      // Watch all fields

// Conditional rendering based on value
{form.watch("accountType") === "business" && (
  <input {...register("companyName")} />
)}
```

### Form Reset
```tsx
// Reset to default values
form.reset();

// Reset to specific values
form.reset({ email: "new@example.com" });

// Reset after successful submit
const onSubmit = async (data: FormData) => {
  await submitForm(data);
  form.reset();
};
```

### Set Values Programmatically
```tsx
// Set single value
form.setValue("email", "preset@example.com");

// Set and trigger validation
form.setValue("email", "preset@example.com", { shouldValidate: true });
```

### Server-Side Errors
```tsx
const onSubmit = async (data: FormData) => {
  const result = await submitToServer(data);
  
  if (result.error) {
    // Set error on specific field
    form.setError("email", {
      type: "server",
      message: result.error,
    });
    
    // Or set root error
    form.setError("root", {
      message: "Something went wrong",
    });
  }
};

// Display root error
{errors.root && <div className="text-red-500">{errors.root.message}</div>}
```

## Validation Timing

```tsx
useForm({
  resolver: zodResolver(schema),
  mode: "onBlur",        // Validate on blur (default: onSubmit)
  reValidateMode: "onChange",  // Re-validate on change after first submit
});
```

| Mode | When it validates |
|------|-------------------|
| `onSubmit` | Only on submit |
| `onBlur` | When field loses focus |
| `onChange` | On every keystroke |
| `onTouched` | On blur, then onChange |
| `all` | On blur and change |

## Gotchas

### Number Inputs
HTML inputs return strings. Coerce to number:
```tsx
const schema = z.object({
  age: z.coerce.number().min(0),
});

<input type="number" {...register("age")} />
```

### Checkbox
```tsx
const schema = z.object({
  acceptTerms: z.literal(true, {
    errorMap: () => ({ message: "You must accept the terms" }),
  }),
});

<input type="checkbox" {...register("acceptTerms")} />
```

### File Input
```tsx
const schema = z.object({
  file: z.instanceof(FileList).refine((files) => files.length > 0, "Required"),
});

<input type="file" {...register("file")} />
```

## Output

Use this pattern as the default for all React forms. Zod schemas are reusable across client and server.
