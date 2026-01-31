# Redis Patterns

Comprehensive guide for Redis caching, data structures, pub/sub, and operational patterns.

## When to Apply

- Implementing caching strategies
- Choosing Redis data structures
- Building pub/sub systems
- Writing Lua scripts for atomic operations
- Configuring Redis Cluster
- Debugging performance issues

## Data Structures & Use Cases

### Strings
```bash
# Basic key-value
SET user:1:name "Alice"
GET user:1:name

# Counters (atomic)
INCR page:views
INCRBY user:1:points 10

# Expiring keys (cache)
SET session:abc123 "user_data" EX 3600  # 1 hour TTL
SETEX cache:api:users 300 "[...]"        # 5 min TTL

# Atomic set-if-not-exists (distributed lock)
SET lock:resource NX EX 30  # Returns OK only if key doesn't exist

# Bit operations (feature flags, bloom filters)
SETBIT user:1:features 0 1  # Enable feature 0
GETBIT user:1:features 0
BITCOUNT user:1:features
```

### Hashes
```bash
# Object storage (more memory efficient than multiple keys)
HSET user:1 name "Alice" email "alice@example.com" age 30
HGET user:1 name
HGETALL user:1
HMGET user:1 name email

# Partial updates
HINCRBY user:1 age 1
HSETNX user:1 created_at "2024-01-01"

# Use case: Session data, user profiles, configurations
```

### Lists
```bash
# Queue (FIFO)
LPUSH queue:jobs '{"task": "email"}'
RPOP queue:jobs                          # Non-blocking
BRPOP queue:jobs 30                      # Blocking with timeout

# Stack (LIFO)
LPUSH stack:undo '{"action": "delete"}'
LPOP stack:undo

# Capped list (recent activity)
LPUSH user:1:activity "viewed product"
LTRIM user:1:activity 0 99               # Keep last 100

# Range queries
LRANGE user:1:activity 0 9               # Last 10 items
```

### Sets
```bash
# Unique collections
SADD tags:post:1 "redis" "caching" "database"
SMEMBERS tags:post:1
SISMEMBER tags:post:1 "redis"

# Set operations
SINTER tags:post:1 tags:post:2           # Common tags
SUNION tags:post:1 tags:post:2           # All tags
SDIFF tags:post:1 tags:post:2            # Tags in 1 but not 2

# Random selection
SRANDMEMBER tags:post:1 3                # 3 random tags
SPOP lottery:entries                     # Random + remove

# Use case: Tags, unique visitors, social connections
```

### Sorted Sets
```bash
# Leaderboards
ZADD leaderboard 1500 "player:1" 1200 "player:2" 1800 "player:3"
ZRANGE leaderboard 0 9 REV WITHSCORES    # Top 10
ZRANK leaderboard "player:1"              # Player's rank
ZINCRBY leaderboard 50 "player:1"         # Add points

# Time-based data
ZADD events 1704067200 "event:1" 1704153600 "event:2"
ZRANGEBYSCORE events 1704000000 1705000000  # Events in range

# Rate limiting (sliding window)
ZADD rate:user:1 <timestamp> <request_id>
ZREMRANGEBYSCORE rate:user:1 0 <timestamp-window>
ZCARD rate:user:1

# Use case: Leaderboards, priority queues, time-series
```

### Streams
```bash
# Event streaming (like Kafka)
XADD events * user_id 1 action "purchase" amount 99.99
XREAD BLOCK 5000 STREAMS events $         # Wait for new events

# Consumer groups
XGROUP CREATE events mygroup $ MKSTREAM
XREADGROUP GROUP mygroup consumer1 COUNT 10 STREAMS events >
XACK events mygroup <message_id>

# Use case: Event sourcing, message queues, activity feeds
```

## Caching Patterns

### Cache-Aside (Lazy Loading)
```python
def get_user(user_id):
    # Try cache first
    cached = redis.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)
    
    # Cache miss: fetch from DB
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    
    # Store in cache with TTL
    redis.setex(f"user:{user_id}", 3600, json.dumps(user))
    return user
```

### Write-Through
```python
def update_user(user_id, data):
    # Update DB first
    db.execute("UPDATE users SET ... WHERE id = ?", data, user_id)
    
    # Update cache
    redis.setex(f"user:{user_id}", 3600, json.dumps(data))
```

### Write-Behind (Async)
```python
def update_user(user_id, data):
    # Update cache immediately
    redis.setex(f"user:{user_id}", 3600, json.dumps(data))
    
    # Queue DB write
    redis.lpush("db:write:queue", json.dumps({
        "table": "users",
        "id": user_id,
        "data": data
    }))
```

### Cache Invalidation
```python
# Tag-based invalidation
def cache_post(post_id, post_data):
    # Cache the post
    redis.setex(f"post:{post_id}", 3600, json.dumps(post_data))
    
    # Track in tag sets
    redis.sadd(f"cache:tag:user:{post_data['user_id']}", f"post:{post_id}")
    for tag in post_data['tags']:
        redis.sadd(f"cache:tag:{tag}", f"post:{post_id}")

def invalidate_tag(tag):
    # Get all keys with this tag
    keys = redis.smembers(f"cache:tag:{tag}")
    if keys:
        redis.delete(*keys)
        redis.delete(f"cache:tag:{tag}")
```

### Thundering Herd Prevention
```python
import random

def get_with_probabilistic_refresh(key, ttl=3600, refresh_threshold=300):
    """Refresh cache before expiration with probability"""
    cached, remaining_ttl = redis.get(key), redis.ttl(key)
    
    if cached and remaining_ttl > refresh_threshold:
        return json.loads(cached)
    
    # Near expiration: probabilistic refresh
    if cached and remaining_ttl > 0:
        # Higher chance of refresh as TTL approaches 0
        refresh_prob = 1 - (remaining_ttl / refresh_threshold)
        if random.random() > refresh_prob:
            return json.loads(cached)
    
    # Acquire lock to prevent thundering herd
    lock_key = f"lock:{key}"
    if redis.set(lock_key, "1", nx=True, ex=30):
        try:
            fresh_data = fetch_from_source()
            redis.setex(key, ttl, json.dumps(fresh_data))
            return fresh_data
        finally:
            redis.delete(lock_key)
    
    # Another process is refreshing, return stale data
    return json.loads(cached) if cached else None
```

## Distributed Locking

### Redlock Algorithm (Single Instance)
```python
import uuid
import time

def acquire_lock(lock_name, timeout=10):
    identifier = str(uuid.uuid4())
    end = time.time() + timeout
    
    while time.time() < end:
        if redis.set(f"lock:{lock_name}", identifier, nx=True, ex=timeout):
            return identifier
        time.sleep(0.001)
    
    return None

def release_lock(lock_name, identifier):
    # Lua script for atomic check-and-delete
    script = """
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
    else
        return 0
    end
    """
    return redis.eval(script, 1, f"lock:{lock_name}", identifier)
```

## Lua Scripting

### Atomic Operations
```lua
-- Rate limiter (sliding window)
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local request_id = ARGV[4]

-- Remove old entries
redis.call('ZREMRANGEBYSCORE', key, 0, now - window)

-- Count current requests
local count = redis.call('ZCARD', key)

if count < limit then
    -- Add request and return success
    redis.call('ZADD', key, now, request_id)
    redis.call('EXPIRE', key, window)
    return 1
else
    return 0
end
```

```python
# Usage
rate_limiter = redis.register_script(lua_script)
allowed = rate_limiter(
    keys=["rate:user:123"],
    args=[100, 60, int(time.time()), str(uuid.uuid4())]
)
```

### Conditional Updates
```lua
-- Optimistic locking
local key = KEYS[1]
local expected_version = tonumber(ARGV[1])
local new_value = ARGV[2]
local new_version = expected_version + 1

local current_version = tonumber(redis.call('HGET', key, 'version') or 0)

if current_version == expected_version then
    redis.call('HSET', key, 'data', new_value, 'version', new_version)
    return new_version
else
    return -1  -- Conflict
end
```

## Pub/Sub Patterns

### Basic Pub/Sub
```python
# Publisher
redis.publish("notifications", json.dumps({
    "type": "new_message",
    "user_id": 123
}))

# Subscriber
pubsub = redis.pubsub()
pubsub.subscribe("notifications")

for message in pubsub.listen():
    if message["type"] == "message":
        data = json.loads(message["data"])
        handle_notification(data)
```

### Pattern Subscriptions
```python
# Subscribe to all user channels
pubsub.psubscribe("user:*:events")

# Publish to specific user
redis.publish("user:123:events", json.dumps({"action": "logout"}))
```

## Cluster Operations

### Key Slot Distribution
```bash
# Keys with same hash tag go to same slot
SET {user:1}:profile "..."
SET {user:1}:settings "..."
# Both keys on same node, can use MULTI/EXEC

# Check slot
CLUSTER KEYSLOT "user:1"
```

### Pipeline in Cluster
```python
# Use hash tags for multi-key operations
pipe = redis.pipeline()
pipe.set("{order:123}:items", "...")
pipe.set("{order:123}:total", "...")
pipe.expire("{order:123}:items", 3600)
pipe.expire("{order:123}:total", 3600)
pipe.execute()  # Single round trip to one node
```

## Performance Tuning

### Memory Optimization
```bash
# Check memory usage
MEMORY USAGE key_name
INFO memory

# Use hashes for small objects (ziplist encoding)
# Configure in redis.conf:
# hash-max-ziplist-entries 512
# hash-max-ziplist-value 64

# Compress large values client-side
# Use msgpack or protobuf instead of JSON
```

### Pipeline Everything
```python
# Bad: N round trips
for user_id in user_ids:
    redis.get(f"user:{user_id}")

# Good: 1 round trip
pipe = redis.pipeline()
for user_id in user_ids:
    pipe.get(f"user:{user_id}")
results = pipe.execute()
```

### Connection Pooling
```python
# Always use connection pools
pool = redis.ConnectionPool(
    host='localhost',
    port=6379,
    max_connections=50,
    decode_responses=True
)
redis_client = redis.Redis(connection_pool=pool)
```

## Common Gotchas

### Key Expiration
- Expiration is per-key, not per-field in hashes
- Use sorted sets with timestamps for field-level expiration
- `EXPIRE` resets on modification unless using `KEEPTTL`

### Memory Leaks
- Always set TTL on cache keys
- Use `SCAN` instead of `KEYS` in production
- Monitor memory with `INFO memory`

### Blocking Commands
- `BLPOP`, `BRPOP` hold connections
- Use separate connection pool for blocking ops
- Set reasonable timeouts

### Cluster Limitations
- Multi-key ops require hash tags
- Lua scripts must use keys on same slot
- No `SELECT` (single database)

## Monitoring Commands

```bash
# Real-time command monitoring
MONITOR                                   # See all commands (careful in prod!)

# Slow query log
SLOWLOG GET 10                            # Last 10 slow queries
CONFIG SET slowlog-log-slower-than 10000  # Log queries > 10ms

# Client connections
CLIENT LIST
CLIENT KILL TYPE normal                   # Kill idle connections

# Memory analysis
MEMORY DOCTOR
MEMORY STATS
DEBUG OBJECT key_name                     # Key details
```

## References

- [Redis Documentation](https://redis.io/docs/)
- [Redis Best Practices](https://redis.io/docs/management/optimization/)
- [Distributed Locks with Redis](https://redis.io/docs/manual/patterns/distributed-locks/)
- [Redis Cluster Tutorial](https://redis.io/docs/management/scaling/)
