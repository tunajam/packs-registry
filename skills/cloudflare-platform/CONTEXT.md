# Cloudflare Platform

Comprehensive guide for Cloudflare Workers, storage services, and edge computing.

## When to Apply

- Building edge applications with Workers
- Choosing between KV, D1, R2, and Durable Objects
- Implementing caching strategies
- Configuring security rules
- Setting up Queues and scheduled tasks
- Deploying with Wrangler

## Workers Basics

### Project Structure
```
my-worker/
├── src/
│   ├── index.ts          # Entry point
│   └── handlers/
├── wrangler.toml         # Configuration
├── package.json
└── tsconfig.json
```

### Minimal Worker
```typescript
export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    return new Response("Hello World!");
  },
};
```

### Request Handling
```typescript
export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    const url = new URL(request.url);
    
    // Routing
    if (url.pathname === "/api/users" && request.method === "GET") {
      return handleGetUsers(request, env);
    }
    
    if (url.pathname.startsWith("/api/users/") && request.method === "GET") {
      const id = url.pathname.split("/").pop();
      return handleGetUser(id, env);
    }
    
    if (request.method === "POST" && url.pathname === "/api/users") {
      const body = await request.json();
      return handleCreateUser(body, env);
    }
    
    return new Response("Not Found", { status: 404 });
  },
};
```

### Environment Bindings
```toml
# wrangler.toml
name = "my-worker"
main = "src/index.ts"
compatibility_date = "2024-01-01"

[vars]
API_URL = "https://api.example.com"

[[kv_namespaces]]
binding = "CACHE"
id = "abc123"

[[r2_buckets]]
binding = "UPLOADS"
bucket_name = "my-uploads"

[[d1_databases]]
binding = "DB"
database_name = "my-database"
database_id = "def456"

[[queues.producers]]
binding = "QUEUE"
queue = "my-queue"

[[durable_objects.bindings]]
name = "COUNTER"
class_name = "Counter"
```

```typescript
interface Env {
  API_URL: string;
  CACHE: KVNamespace;
  UPLOADS: R2Bucket;
  DB: D1Database;
  QUEUE: Queue;
  COUNTER: DurableObjectNamespace;
}
```

## KV (Key-Value Store)

### Use Cases
- Session storage
- Feature flags
- Configuration
- Caching (read-heavy)

### Operations
```typescript
// Write (eventually consistent, ~60s)
await env.CACHE.put("key", "value");
await env.CACHE.put("key", JSON.stringify(data));
await env.CACHE.put("key", "value", { expirationTtl: 3600 });
await env.CACHE.put("key", "value", { metadata: { version: 1 } });

// Read
const value = await env.CACHE.get("key");
const data = await env.CACHE.get("key", { type: "json" });
const { value, metadata } = await env.CACHE.getWithMetadata("key");

// Delete
await env.CACHE.delete("key");

// List
const list = await env.CACHE.list({ prefix: "user:", limit: 100 });
for (const key of list.keys) {
  console.log(key.name, key.metadata);
}
```

### KV Limitations
- Eventually consistent (60s propagation)
- Max value size: 25 MB
- Max key size: 512 bytes
- Not for write-heavy workloads

## R2 (Object Storage)

### Use Cases
- File uploads
- Static assets
- Large objects
- S3-compatible storage

### Operations
```typescript
// Upload
await env.UPLOADS.put("files/doc.pdf", request.body, {
  httpMetadata: { contentType: "application/pdf" },
  customMetadata: { userId: "123" }
});

// Download
const object = await env.UPLOADS.get("files/doc.pdf");
if (object) {
  return new Response(object.body, {
    headers: {
      "Content-Type": object.httpMetadata?.contentType || "application/octet-stream",
      "ETag": object.etag
    }
  });
}

// List
const objects = await env.UPLOADS.list({ prefix: "files/", limit: 100 });
for (const object of objects.objects) {
  console.log(object.key, object.size);
}

// Delete
await env.UPLOADS.delete("files/doc.pdf");
await env.UPLOADS.delete(["file1.pdf", "file2.pdf"]); // Batch

// Multipart upload
const multipart = await env.UPLOADS.createMultipartUpload("large-file.zip");
const part1 = await multipart.uploadPart(1, chunk1);
const part2 = await multipart.uploadPart(2, chunk2);
await multipart.complete([part1, part2]);
```

### Presigned URLs
```typescript
// Generate presigned URL for direct upload
const url = await env.UPLOADS.createPresignedUrl("uploads/file.pdf", {
  method: "PUT",
  expiresIn: 3600
});
```

## D1 (SQLite Database)

### Use Cases
- Relational data
- ACID transactions
- Complex queries
- Auth/session data

### Schema & Migrations
```sql
-- migrations/0001_initial.sql
CREATE TABLE users (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  email TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
```

```bash
# Run migrations
wrangler d1 migrations apply my-database
```

### Queries
```typescript
// Simple query
const result = await env.DB.prepare(
  "SELECT * FROM users WHERE id = ?"
).bind(userId).first();

// Multiple results
const { results } = await env.DB.prepare(
  "SELECT * FROM users WHERE created_at > ?"
).bind(date).all();

// Insert
const { meta } = await env.DB.prepare(
  "INSERT INTO users (email, name) VALUES (?, ?)"
).bind(email, name).run();
const newId = meta.last_row_id;

// Batch operations
const batch = await env.DB.batch([
  env.DB.prepare("INSERT INTO users (email, name) VALUES (?, ?)").bind(email1, name1),
  env.DB.prepare("INSERT INTO users (email, name) VALUES (?, ?)").bind(email2, name2),
]);

// Transaction-like behavior (all or nothing)
```

### D1 Best Practices
- Use prepared statements (SQL injection protection)
- Batch related queries
- Create indexes for query patterns
- Keep database size reasonable (< 10GB)

## Durable Objects

### Use Cases
- Real-time collaboration
- WebSocket connections
- Counters/rate limiting
- Stateful coordination
- Game servers

### Implementation
```typescript
// Durable Object class
export class Counter {
  state: DurableObjectState;
  value: number = 0;

  constructor(state: DurableObjectState, env: Env) {
    this.state = state;
    this.state.blockConcurrencyWhile(async () => {
      this.value = await this.state.storage.get("value") || 0;
    });
  }

  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);
    
    if (url.pathname === "/increment") {
      this.value++;
      await this.state.storage.put("value", this.value);
    }
    
    return new Response(JSON.stringify({ value: this.value }));
  }
}

// Worker calling Durable Object
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Get DO stub by ID
    const id = env.COUNTER.idFromName("global-counter");
    const stub = env.COUNTER.get(id);
    
    // Forward request
    return stub.fetch(request);
  }
};
```

### WebSocket with Durable Objects
```typescript
export class ChatRoom {
  sessions: Map<WebSocket, { username: string }> = new Map();
  
  async fetch(request: Request): Promise<Response> {
    if (request.headers.get("Upgrade") === "websocket") {
      const [client, server] = Object.values(new WebSocketPair());
      
      server.accept();
      this.sessions.set(server, { username: "anonymous" });
      
      server.addEventListener("message", (event) => {
        this.broadcast(event.data);
      });
      
      server.addEventListener("close", () => {
        this.sessions.delete(server);
      });
      
      return new Response(null, { status: 101, webSocket: client });
    }
    
    return new Response("Expected WebSocket", { status: 400 });
  }
  
  broadcast(message: string) {
    for (const [ws] of this.sessions) {
      ws.send(message);
    }
  }
}
```

## Queues

### Producer
```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Send single message
    await env.QUEUE.send({ type: "email", to: "user@example.com" });
    
    // Send batch
    await env.QUEUE.sendBatch([
      { body: { type: "email", to: "user1@example.com" } },
      { body: { type: "email", to: "user2@example.com" } },
    ]);
    
    return new Response("Queued!");
  }
};
```

### Consumer
```typescript
export default {
  async queue(batch: MessageBatch<any>, env: Env): Promise<void> {
    for (const message of batch.messages) {
      try {
        await processMessage(message.body, env);
        message.ack();
      } catch (error) {
        message.retry();
      }
    }
  }
};
```

```toml
# wrangler.toml
[[queues.consumers]]
queue = "my-queue"
max_batch_size = 10
max_batch_timeout = 30
max_retries = 3
dead_letter_queue = "my-dlq"
```

## Scheduled Workers (Cron)

```typescript
export default {
  async scheduled(event: ScheduledEvent, env: Env, ctx: ExecutionContext): Promise<void> {
    switch (event.cron) {
      case "0 * * * *": // Hourly
        await hourlyTask(env);
        break;
      case "0 0 * * *": // Daily
        await dailyTask(env);
        break;
    }
  }
};
```

```toml
# wrangler.toml
[triggers]
crons = ["0 * * * *", "0 0 * * *"]
```

## Caching

### Cache API
```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const cache = caches.default;
    const cacheKey = new Request(request.url, request);
    
    // Check cache
    let response = await cache.match(cacheKey);
    if (response) {
      return response;
    }
    
    // Fetch and cache
    response = await fetch(request);
    
    if (response.ok) {
      const responseToCache = new Response(response.body, response);
      responseToCache.headers.set("Cache-Control", "public, max-age=3600");
      
      // Cache in background
      ctx.waitUntil(cache.put(cacheKey, responseToCache.clone()));
    }
    
    return response;
  }
};
```

### Cache Tags (Enterprise)
```typescript
const response = new Response(body, {
  headers: {
    "Cache-Tag": "product-123, category-electronics",
    "Cache-Control": "public, max-age=3600"
  }
});
```

## Security

### WAF Rules
```javascript
// Managed challenge for suspicious requests
if (request.cf.botManagement.score < 30) {
  return new Response("Verification required", { status: 403 });
}

// Rate limiting (requires Rate Limiting rules)
// Configure in dashboard or via API
```

### JWT Validation
```typescript
import { verify } from "@tsndr/cloudflare-worker-jwt";

async function validateToken(token: string, secret: string): Promise<boolean> {
  return await verify(token, secret);
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const authHeader = request.headers.get("Authorization");
    if (!authHeader?.startsWith("Bearer ")) {
      return new Response("Unauthorized", { status: 401 });
    }
    
    const token = authHeader.slice(7);
    const isValid = await validateToken(token, env.JWT_SECRET);
    
    if (!isValid) {
      return new Response("Invalid token", { status: 401 });
    }
    
    // Continue with authenticated request
  }
};
```

## Wrangler CLI

```bash
# Development
wrangler dev                    # Start dev server
wrangler dev --remote          # Dev with remote bindings

# Deployment
wrangler deploy                 # Deploy to production
wrangler deploy --env staging   # Deploy to staging

# Secrets
wrangler secret put API_KEY     # Set secret
wrangler secret list            # List secrets

# KV
wrangler kv:namespace create CACHE
wrangler kv:key put --binding=CACHE "key" "value"
wrangler kv:key get --binding=CACHE "key"

# D1
wrangler d1 create my-database
wrangler d1 migrations create my-database initial
wrangler d1 migrations apply my-database

# R2
wrangler r2 bucket create my-bucket
wrangler r2 object put my-bucket/file.txt --file ./file.txt

# Logs
wrangler tail                   # Stream logs
```

## Common Patterns

### Error Handling
```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    try {
      return await handleRequest(request, env);
    } catch (error) {
      console.error("Error:", error);
      
      if (error instanceof ValidationError) {
        return new Response(JSON.stringify({ error: error.message }), {
          status: 400,
          headers: { "Content-Type": "application/json" }
        });
      }
      
      return new Response("Internal Server Error", { status: 500 });
    }
  }
};
```

### CORS
```typescript
const corsHeaders = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Methods": "GET, POST, PUT, DELETE, OPTIONS",
  "Access-Control-Allow-Headers": "Content-Type, Authorization",
};

function handleOptions(request: Request): Response {
  return new Response(null, { headers: corsHeaders });
}

function addCorsHeaders(response: Response): Response {
  const newResponse = new Response(response.body, response);
  for (const [key, value] of Object.entries(corsHeaders)) {
    newResponse.headers.set(key, value);
  }
  return newResponse;
}
```

## Limitations

| Service | Limit |
|---------|-------|
| Worker CPU time | 10-50ms (free), 30s (paid) |
| Worker memory | 128 MB |
| KV value size | 25 MB |
| KV reads/worker | 1000/request |
| D1 database size | 10 GB |
| D1 rows/query | 10,000 |
| R2 object size | 5 TB |
| Queue message size | 128 KB |
| Durable Object storage | 1 GB |

## References

- [Workers Documentation](https://developers.cloudflare.com/workers/)
- [D1 Documentation](https://developers.cloudflare.com/d1/)
- [R2 Documentation](https://developers.cloudflare.com/r2/)
- [Durable Objects](https://developers.cloudflare.com/durable-objects/)
- [Queues](https://developers.cloudflare.com/queues/)
