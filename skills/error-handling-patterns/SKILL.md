# Skill: Error Handling Patterns

Consistent patterns for handling errors across different languages and contexts.

## When to Use
- Designing error handling for a new project
- Reviewing code for error handling gaps
- Deciding what to catch vs let bubble
- Creating user-facing error messages

## Core Principles

### 1. Fail Fast, Fail Loud
Detect errors early. Don't let bad state propagate.

```javascript
// ❌ Silent failure
function getUser(id) {
  const user = db.find(id);
  return user || {};  // Returns empty object, caller doesn't know it failed
}

// ✅ Explicit failure
function getUser(id) {
  const user = db.find(id);
  if (!user) throw new NotFoundError(`User ${id} not found`);
  return user;
}
```

### 2. Handle at the Right Level
Only catch errors you can actually handle.

```javascript
// ❌ Catching everything, handling nothing
try {
  doSomething();
} catch (e) {
  console.log(e);  // Now what?
}

// ✅ Handle what you can, let the rest bubble
try {
  await fetchData();
} catch (e) {
  if (e instanceof NetworkError) {
    return cachedData;  // Actual recovery
  }
  throw e;  // Can't handle this, let it go up
}
```

### 3. Errors Are Data
Errors should be typed, structured, and useful.

```typescript
// ❌ Stringly typed
throw new Error("validation failed");

// ✅ Structured error
class ValidationError extends Error {
  constructor(
    public field: string,
    public reason: string,
    public value: unknown
  ) {
    super(`${field}: ${reason}`);
    this.name = 'ValidationError';
  }
}

throw new ValidationError('email', 'invalid format', 'not-an-email');
```

## Error Categories

### Recoverable (Handle and Continue)
- Network timeout → Retry
- File locked → Wait and retry
- Invalid input → Show validation error

### Non-Recoverable (Crash and Report)
- Out of memory → Crash
- Corrupted data → Crash
- Missing critical config → Crash

### Partial Failure (Degrade Gracefully)
- Recommendation engine down → Show popular items
- Avatar service down → Show initials
- Analytics failed → Log it, continue

## Language-Specific Patterns

### TypeScript/JavaScript

```typescript
// Result type (explicit success/failure)
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };

async function fetchUser(id: string): Promise<Result<User, FetchError>> {
  try {
    const user = await api.get(`/users/${id}`);
    return { ok: true, value: user };
  } catch (e) {
    return { ok: false, error: new FetchError(e) };
  }
}

// Usage
const result = await fetchUser(id);
if (!result.ok) {
  showError(result.error.message);
  return;
}
doSomething(result.value);
```

### Go

```go
// Errors are values, always check them
user, err := getUser(id)
if err != nil {
    if errors.Is(err, ErrNotFound) {
        return nil, fmt.Errorf("user %s: %w", id, err)
    }
    return nil, err
}

// Wrap errors with context
if err := db.Save(user); err != nil {
    return fmt.Errorf("saving user %s: %w", user.ID, err)
}
```

### Python

```python
# Custom exception hierarchy
class AppError(Exception):
    """Base for all app errors"""
    pass

class ValidationError(AppError):
    def __init__(self, field: str, message: str):
        self.field = field
        super().__init__(f"{field}: {message}")

class NotFoundError(AppError):
    pass

# Use specific exceptions
try:
    user = get_user(id)
except NotFoundError:
    return {"error": "User not found"}, 404
except ValidationError as e:
    return {"error": str(e), "field": e.field}, 400
```

## API Error Responses

### Standard Format
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Email is required",
    "field": "email",
    "details": {}
  }
}
```

### HTTP Status Codes
| Code | Use For |
|------|---------|
| 400 | Bad input (validation, malformed JSON) |
| 401 | Not authenticated |
| 403 | Authenticated but not authorized |
| 404 | Resource doesn't exist |
| 409 | Conflict (duplicate, already exists) |
| 422 | Valid syntax but can't process |
| 429 | Rate limited |
| 500 | Server broke, not client's fault |

### Don't Leak Internals
```javascript
// ❌ Exposes implementation
{ "error": "SQLITE_CONSTRAINT: UNIQUE constraint failed: users.email" }

// ✅ User-friendly, safe
{ "error": { "code": "EMAIL_EXISTS", "message": "This email is already registered" } }
```

## User-Facing Messages

### Principles
- **Say what happened** — "Your session expired"
- **Say what to do** — "Please log in again"
- **Don't blame** — Not "You entered wrong password" but "Password didn't match"
- **Be specific** — Not "An error occurred" but "Couldn't save your changes"

### Examples
```
❌ "Error 500"
✅ "Something went wrong on our end. Please try again in a few minutes."

❌ "Invalid input"
✅ "Email address doesn't look right. Check for typos?"

❌ "Operation failed"
✅ "Couldn't upload your photo. It might be too large (max 5MB)."
```

## Logging Errors

```javascript
// Include context
logger.error('Failed to process payment', {
  error: e.message,
  stack: e.stack,
  userId: user.id,
  amount: payment.amount,
  paymentId: payment.id
});
```

### Log Levels
- **error** — Something broke, needs attention
- **warn** — Suspicious but recovered (rate limits, retries)
- **info** — Normal operations (user logged in)
- **debug** — Detailed debugging info

## Output

When implementing error handling:
1. Define your error types/hierarchy
2. Decide what's recoverable vs fatal
3. Ensure errors have context (what, where, why)
4. User messages are helpful, not technical
5. Logs include enough info to debug
