# API Design Patterns Context

Comprehensive guide to REST API design, error handling, and best practices.

## URL Design

### Resource Naming
```
# Resources are nouns, plural
GET    /users              # List users
POST   /users              # Create user
GET    /users/{id}         # Get user
PUT    /users/{id}         # Replace user
PATCH  /users/{id}         # Update user
DELETE /users/{id}         # Delete user

# Nested resources
GET    /users/{id}/posts   # User's posts
POST   /users/{id}/posts   # Create user's post
GET    /posts/{id}/comments # Post's comments

# Actions as resources (when needed)
POST   /users/{id}/activate      # User action
POST   /orders/{id}/cancel       # Order action
POST   /auth/login               # Auth action
POST   /auth/logout              # Auth action
```

### Query Parameters
```
# Filtering
GET /users?role=admin
GET /users?role=admin&status=active
GET /posts?author=123&published=true

# Sorting
GET /users?sort=createdAt:desc
GET /users?sort=name:asc,createdAt:desc

# Pagination
GET /users?page=2&limit=20
GET /users?offset=40&limit=20
GET /users?cursor=abc123&limit=20

# Field selection
GET /users?fields=id,name,email
GET /users?include=posts,comments

# Search
GET /users?q=john
GET /users?search=john@example
```

## HTTP Methods

| Method | Use Case | Idempotent | Safe |
|--------|----------|------------|------|
| GET | Retrieve resource | Yes | Yes |
| POST | Create resource | No | No |
| PUT | Replace resource | Yes | No |
| PATCH | Partial update | No | No |
| DELETE | Remove resource | Yes | No |
| HEAD | Get headers only | Yes | Yes |
| OPTIONS | Get allowed methods | Yes | Yes |

## Status Codes

### Success (2xx)
```
200 OK              - Request succeeded (GET, PUT, PATCH)
201 Created         - Resource created (POST)
202 Accepted        - Request accepted for processing
204 No Content      - Success with no response body (DELETE)
```

### Redirection (3xx)
```
301 Moved Permanently    - Resource moved permanently
302 Found                - Temporary redirect
304 Not Modified         - Cached version valid
```

### Client Error (4xx)
```
400 Bad Request          - Invalid request body/params
401 Unauthorized         - Authentication required
403 Forbidden            - Authenticated but not authorized
404 Not Found            - Resource doesn't exist
405 Method Not Allowed   - HTTP method not supported
409 Conflict             - Resource conflict (duplicate)
410 Gone                 - Resource permanently deleted
422 Unprocessable Entity - Validation error
429 Too Many Requests    - Rate limit exceeded
```

### Server Error (5xx)
```
500 Internal Server Error - Unexpected error
502 Bad Gateway           - Upstream service error
503 Service Unavailable   - Server temporarily unavailable
504 Gateway Timeout       - Upstream timeout
```

## Request/Response Format

### Request
```typescript
// Content-Type: application/json
interface CreateUserRequest {
  name: string
  email: string
  password: string
  role?: 'user' | 'admin'
}

// Headers
Authorization: Bearer <token>
Content-Type: application/json
Accept: application/json
X-Request-ID: <uuid>
```

### Response
```typescript
// Single resource
interface UserResponse {
  id: string
  name: string
  email: string
  role: string
  createdAt: string
  updatedAt: string
}

// Collection
interface UsersResponse {
  data: UserResponse[]
  meta: {
    total: number
    page: number
    limit: number
    totalPages: number
  }
}

// With pagination links
interface PaginatedResponse<T> {
  data: T[]
  meta: {
    total: number
    page: number
    limit: number
  }
  links: {
    self: string
    first: string
    prev: string | null
    next: string | null
    last: string
  }
}
```

## Error Response Format

```typescript
interface ErrorResponse {
  error: {
    code: string           // Machine-readable code
    message: string        // Human-readable message
    details?: ErrorDetail[] // Validation errors
    requestId?: string     // For debugging
    timestamp?: string     // When error occurred
  }
}

interface ErrorDetail {
  field: string
  message: string
  code: string
}

// Examples
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request data",
    "details": [
      { "field": "email", "message": "Invalid email format", "code": "INVALID_FORMAT" },
      { "field": "password", "message": "Must be at least 8 characters", "code": "TOO_SHORT" }
    ],
    "requestId": "req_abc123",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}

{
  "error": {
    "code": "NOT_FOUND",
    "message": "User not found",
    "requestId": "req_def456"
  }
}

{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Too many requests. Please try again in 60 seconds.",
    "requestId": "req_ghi789"
  }
}
```

## Error Code Constants

```typescript
// Common error codes
const ErrorCodes = {
  // Validation
  VALIDATION_ERROR: 'VALIDATION_ERROR',
  INVALID_FORMAT: 'INVALID_FORMAT',
  REQUIRED_FIELD: 'REQUIRED_FIELD',
  
  // Authentication
  UNAUTHORIZED: 'UNAUTHORIZED',
  INVALID_TOKEN: 'INVALID_TOKEN',
  TOKEN_EXPIRED: 'TOKEN_EXPIRED',
  
  // Authorization
  FORBIDDEN: 'FORBIDDEN',
  INSUFFICIENT_PERMISSIONS: 'INSUFFICIENT_PERMISSIONS',
  
  // Resources
  NOT_FOUND: 'NOT_FOUND',
  ALREADY_EXISTS: 'ALREADY_EXISTS',
  CONFLICT: 'CONFLICT',
  
  // Rate limiting
  RATE_LIMITED: 'RATE_LIMITED',
  
  // Server
  INTERNAL_ERROR: 'INTERNAL_ERROR',
  SERVICE_UNAVAILABLE: 'SERVICE_UNAVAILABLE',
} as const
```

## Versioning

### URL Path (Recommended)
```
/v1/users
/v2/users
```

### Header
```
Accept: application/vnd.api.v1+json
API-Version: 1
```

### Query Parameter
```
/users?version=1
```

## Authentication

### Bearer Token
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

### API Key
```
X-API-Key: your-api-key
# Or in query (less secure)
/users?api_key=your-api-key
```

## Rate Limiting Headers

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1704067200
Retry-After: 60
```

## HATEOAS Links

```json
{
  "id": "123",
  "name": "John",
  "_links": {
    "self": { "href": "/users/123" },
    "posts": { "href": "/users/123/posts" },
    "edit": { "href": "/users/123", "method": "PATCH" },
    "delete": { "href": "/users/123", "method": "DELETE" }
  }
}
```

## Filtering Best Practices

```
# Equality
?status=active

# Comparison
?price[gte]=100&price[lte]=500
?createdAt[gt]=2024-01-01

# Multiple values
?status=active,pending
?status[]=active&status[]=pending

# Nested fields
?author.name=John

# Full-text search
?q=search+term
?search=john
```

## Common Patterns

### Bulk Operations
```
# Bulk create
POST /users/bulk
Body: { "users": [...] }

# Bulk update
PATCH /users/bulk
Body: { "ids": ["1", "2"], "data": { "status": "active" } }

# Bulk delete
DELETE /users/bulk
Body: { "ids": ["1", "2", "3"] }
```

### Soft Delete
```
DELETE /users/123           # Soft delete
GET /users/123              # Returns 404 or deleted resource
GET /users?includeDeleted=true  # Include deleted
```

### Async Operations
```
POST /reports
Response: 202 Accepted
{
  "id": "job-123",
  "status": "processing",
  "_links": {
    "status": "/jobs/job-123",
    "result": "/reports/job-123"
  }
}

GET /jobs/job-123
{
  "id": "job-123",
  "status": "completed",
  "progress": 100,
  "resultUrl": "/reports/job-123"
}
```

## Best Practices

1. **Use nouns for resources** - `/users` not `/getUsers`
2. **Use plural names** - `/users` not `/user`
3. **Keep URLs flat** - Avoid deep nesting
4. **Use consistent naming** - camelCase or snake_case
5. **Return consistent response format** - Same structure everywhere
6. **Include useful error messages** - Help clients debug
7. **Version your API** - Plan for breaking changes
8. **Document everything** - OpenAPI/Swagger
9. **Use proper status codes** - Not just 200 and 500
10. **Implement rate limiting** - Protect your API
