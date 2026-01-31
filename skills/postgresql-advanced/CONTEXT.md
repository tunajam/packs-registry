# PostgreSQL Advanced

Advanced PostgreSQL patterns for queries, indexes, performance tuning, and operations.

## When to Apply

- Optimizing slow queries
- Designing indexes
- Writing complex queries (CTEs, window functions)
- Troubleshooting performance issues
- Using PostgreSQL extensions
- Managing migrations and schema changes

## Index Strategies

### Index Types
```sql
-- B-tree (default, most common)
CREATE INDEX idx_users_email ON users(email);

-- Partial index (filtered)
CREATE INDEX idx_active_users ON users(email) WHERE active = true;

-- Composite index (column order matters!)
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at DESC);

-- Covering index (includes additional columns)
CREATE INDEX idx_orders_covering ON orders(user_id) INCLUDE (total, status);

-- Expression index
CREATE INDEX idx_users_email_lower ON users(lower(email));

-- GIN (for arrays, JSONB, full-text)
CREATE INDEX idx_posts_tags ON posts USING gin(tags);
CREATE INDEX idx_data_jsonb ON data USING gin(metadata jsonb_path_ops);

-- GiST (for geometric, range types)
CREATE INDEX idx_locations_point ON locations USING gist(coordinates);

-- BRIN (for large, naturally ordered tables)
CREATE INDEX idx_logs_time ON logs USING brin(created_at);
```

### Index Selection Guidelines
| Query Pattern | Index Type |
|---------------|------------|
| Equality (`=`) | B-tree |
| Range (`<`, `>`, `BETWEEN`) | B-tree |
| Pattern (`LIKE 'foo%'`) | B-tree |
| Pattern (`LIKE '%foo%'`) | GIN + pg_trgm |
| Array contains (`@>`) | GIN |
| JSONB queries | GIN |
| Full-text search | GIN |
| Time-series (sequential) | BRIN |
| Spatial queries | GiST |

### Composite Index Column Order
```sql
-- Query: WHERE user_id = 1 AND created_at > '2024-01-01'
-- Good: (user_id, created_at) - equality column first
CREATE INDEX idx_good ON orders(user_id, created_at);

-- Bad: (created_at, user_id) - range column first
-- This index can only use created_at, not user_id efficiently
```

## Query Optimization

### EXPLAIN ANALYZE
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE user_id = 123 AND status = 'pending';

-- Key metrics to watch:
-- - Seq Scan on large tables (needs index)
-- - High "actual time" vs "cost" 
-- - Buffers: shared hit vs read (cache misses)
-- - Rows removed by filter (index not selective enough)
```

### Common Query Patterns

#### Pagination (Offset vs Keyset)
```sql
-- Bad: OFFSET scales poorly
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 10000;

-- Good: Keyset pagination
SELECT * FROM posts 
WHERE created_at < '2024-01-15 10:00:00'
ORDER BY created_at DESC 
LIMIT 20;

-- With composite cursor
SELECT * FROM posts 
WHERE (created_at, id) < ('2024-01-15 10:00:00', 12345)
ORDER BY created_at DESC, id DESC 
LIMIT 20;
```

#### Avoid SELECT *
```sql
-- Bad: fetches all columns
SELECT * FROM users WHERE id = 1;

-- Good: only needed columns
SELECT id, email, name FROM users WHERE id = 1;
```

#### EXISTS vs IN
```sql
-- Better for subqueries that might return many rows
SELECT * FROM orders o
WHERE EXISTS (SELECT 1 FROM vip_users v WHERE v.id = o.user_id);

-- vs
SELECT * FROM orders WHERE user_id IN (SELECT id FROM vip_users);
```

#### Batch Operations
```sql
-- Bad: N individual INSERTs
INSERT INTO items (name) VALUES ('a');
INSERT INTO items (name) VALUES ('b');

-- Good: Single multi-row INSERT
INSERT INTO items (name) VALUES ('a'), ('b'), ('c');

-- Or COPY for bulk loads
COPY items (name) FROM '/path/to/file.csv' WITH CSV;
```

## Window Functions

```sql
-- Row number (for pagination alternatives)
SELECT 
  *,
  ROW_NUMBER() OVER (ORDER BY created_at DESC) as row_num
FROM posts;

-- Running total
SELECT 
  order_id,
  amount,
  SUM(amount) OVER (ORDER BY created_at) as running_total
FROM orders;

-- Rank within groups
SELECT 
  user_id,
  product_id,
  amount,
  RANK() OVER (PARTITION BY user_id ORDER BY amount DESC) as rank
FROM purchases;

-- Lead/lag (previous/next row)
SELECT 
  created_at,
  revenue,
  revenue - LAG(revenue) OVER (ORDER BY created_at) as change_from_prev
FROM daily_revenue;

-- Moving average
SELECT 
  date,
  value,
  AVG(value) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) as moving_avg_7d
FROM metrics;
```

## CTEs (Common Table Expressions)

### Basic CTE
```sql
WITH active_users AS (
  SELECT id, email FROM users WHERE active = true
)
SELECT u.email, COUNT(o.id) as order_count
FROM active_users u
JOIN orders o ON o.user_id = u.id
GROUP BY u.email;
```

### Recursive CTE
```sql
-- Hierarchical data (org chart, categories)
WITH RECURSIVE subordinates AS (
  -- Base case
  SELECT id, name, manager_id, 0 as level
  FROM employees
  WHERE id = 1  -- Start from CEO
  
  UNION ALL
  
  -- Recursive case
  SELECT e.id, e.name, e.manager_id, s.level + 1
  FROM employees e
  JOIN subordinates s ON e.manager_id = s.id
)
SELECT * FROM subordinates;
```

### Writeable CTE
```sql
-- Archive and delete in one query
WITH archived AS (
  DELETE FROM events
  WHERE created_at < NOW() - INTERVAL '1 year'
  RETURNING *
)
INSERT INTO events_archive SELECT * FROM archived;
```

## JSONB Operations

### Query JSONB
```sql
-- Access nested value
SELECT data->'user'->>'name' FROM records;
SELECT data->>'name' FROM records;  -- text
SELECT data->'age' FROM records;     -- jsonb

-- Where conditions
SELECT * FROM records WHERE data->>'status' = 'active';
SELECT * FROM records WHERE data->'tags' ? 'important';  -- contains key
SELECT * FROM records WHERE data @> '{"status": "active"}';  -- contains

-- Array in JSONB
SELECT * FROM records WHERE data->'tags' @> '["urgent"]';
```

### Update JSONB
```sql
-- Set nested value
UPDATE records SET data = jsonb_set(data, '{user,name}', '"New Name"');

-- Remove key
UPDATE records SET data = data - 'old_field';

-- Merge objects
UPDATE records SET data = data || '{"new_field": "value"}';
```

### JSONB Indexes
```sql
-- General purpose (supports @>, ?, ?|, ?&)
CREATE INDEX idx_data ON records USING gin(data);

-- Path-specific (faster for specific queries)
CREATE INDEX idx_data_status ON records USING btree((data->>'status'));

-- jsonb_path_ops (faster @>, smaller index)
CREATE INDEX idx_data_pathops ON records USING gin(data jsonb_path_ops);
```

## Full-Text Search

### Setup
```sql
-- Add tsvector column
ALTER TABLE articles ADD COLUMN search_vector tsvector;

-- Populate
UPDATE articles SET search_vector = 
  to_tsvector('english', coalesce(title, '') || ' ' || coalesce(body, ''));

-- Index
CREATE INDEX idx_articles_search ON articles USING gin(search_vector);

-- Keep updated with trigger
CREATE TRIGGER articles_search_update
BEFORE INSERT OR UPDATE ON articles
FOR EACH ROW EXECUTE FUNCTION
  tsvector_update_trigger(search_vector, 'pg_catalog.english', title, body);
```

### Search
```sql
-- Basic search
SELECT * FROM articles
WHERE search_vector @@ to_tsquery('english', 'postgres & performance');

-- With ranking
SELECT 
  title,
  ts_rank(search_vector, query) as rank
FROM articles, to_tsquery('english', 'postgres & performance') query
WHERE search_vector @@ query
ORDER BY rank DESC;

-- Phrase search
SELECT * FROM articles
WHERE search_vector @@ phraseto_tsquery('english', 'database performance');

-- Highlighting
SELECT 
  ts_headline('english', body, to_tsquery('postgres'), 'StartSel=<b>, StopSel=</b>')
FROM articles
WHERE search_vector @@ to_tsquery('postgres');
```

## Extensions

### pg_stat_statements (Query Analysis)
```sql
CREATE EXTENSION pg_stat_statements;

-- Find slow queries
SELECT 
  query,
  calls,
  mean_exec_time,
  total_exec_time,
  rows
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;

-- Reset stats
SELECT pg_stat_statements_reset();
```

### pg_trgm (Fuzzy Search)
```sql
CREATE EXTENSION pg_trgm;

-- Similarity search
SELECT name, similarity(name, 'PostgreSLQ') as sim
FROM products
WHERE similarity(name, 'PostgreSLQ') > 0.3
ORDER BY sim DESC;

-- Fuzzy index
CREATE INDEX idx_name_trgm ON products USING gin(name gin_trgm_ops);

-- LIKE/ILIKE with index
SELECT * FROM products WHERE name ILIKE '%database%';
```

### uuid-ossp / pgcrypto
```sql
CREATE EXTENSION "uuid-ossp";
CREATE EXTENSION pgcrypto;

-- Generate UUIDs
SELECT uuid_generate_v4();
SELECT gen_random_uuid();  -- pgcrypto, faster

-- Default UUID primary key
CREATE TABLE items (
  id uuid DEFAULT gen_random_uuid() PRIMARY KEY,
  name text
);
```

## Performance Monitoring

### Table Statistics
```sql
-- Table size
SELECT 
  relname as table,
  pg_size_pretty(pg_total_relation_size(relid)) as total_size,
  pg_size_pretty(pg_relation_size(relid)) as table_size,
  pg_size_pretty(pg_indexes_size(relid)) as index_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;

-- Row estimates
SELECT relname, reltuples::bigint as row_estimate
FROM pg_class
WHERE relkind = 'r' AND relnamespace = 'public'::regnamespace;
```

### Index Usage
```sql
-- Unused indexes
SELECT 
  schemaname || '.' || relname as table,
  indexrelname as index,
  pg_size_pretty(pg_relation_size(i.indexrelid)) as size,
  idx_scan as scans
FROM pg_stat_user_indexes ui
JOIN pg_index i ON ui.indexrelid = i.indexrelid
WHERE NOT indisunique AND idx_scan < 50
ORDER BY pg_relation_size(i.indexrelid) DESC;

-- Index hit ratio (should be > 0.99)
SELECT 
  sum(idx_blks_hit) / nullif(sum(idx_blks_hit + idx_blks_read), 0) as ratio
FROM pg_statio_user_indexes;
```

### Connection Stats
```sql
-- Active connections
SELECT 
  datname,
  usename,
  state,
  query,
  now() - query_start as duration
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;

-- Kill long-running query
SELECT pg_terminate_backend(pid);
```

## Migrations Best Practices

### Safe Schema Changes
```sql
-- Adding column (always safe)
ALTER TABLE users ADD COLUMN bio text;

-- Adding column with default (safe in PG 11+)
ALTER TABLE users ADD COLUMN verified boolean DEFAULT false;

-- Adding index concurrently (doesn't lock table)
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);

-- Renaming column (brief lock, but quick)
ALTER TABLE users RENAME COLUMN old_name TO new_name;
```

### Unsafe Operations (Require Care)
```sql
-- Adding NOT NULL to existing column
-- 1. Add column nullable
-- 2. Backfill data
-- 3. Add constraint with NOT VALID
-- 4. Validate in separate transaction

ALTER TABLE users ADD COLUMN status text;
UPDATE users SET status = 'active' WHERE status IS NULL;
ALTER TABLE users ADD CONSTRAINT users_status_not_null 
  CHECK (status IS NOT NULL) NOT VALID;
ALTER TABLE users VALIDATE CONSTRAINT users_status_not_null;

-- Changing column type (can lock and rewrite)
-- Consider: add new column, migrate, drop old
```

## Backup & Recovery

```bash
# Logical backup (portable)
pg_dump -Fc dbname > backup.dump
pg_dump --schema-only dbname > schema.sql
pg_dump --data-only dbname > data.sql

# Restore
pg_restore -d dbname backup.dump

# Copy table between databases
pg_dump -t tablename source_db | psql target_db

# Point-in-time recovery (requires WAL archiving)
# Configure in postgresql.conf:
# archive_mode = on
# archive_command = 'cp %p /archive/%f'
```

## Configuration Tuning

```ini
# postgresql.conf key settings

# Memory
shared_buffers = '4GB'           # 25% of RAM
effective_cache_size = '12GB'    # 75% of RAM
work_mem = '256MB'               # Per-operation memory
maintenance_work_mem = '1GB'     # For VACUUM, CREATE INDEX

# WAL
wal_buffers = '64MB'
checkpoint_completion_target = 0.9

# Query planner
random_page_cost = 1.1           # For SSDs (default 4.0)
effective_io_concurrency = 200   # For SSDs

# Connections
max_connections = 200
```

## References

- [PostgreSQL Documentation](https://www.postgresql.org/docs/current/)
- [Use The Index, Luke](https://use-the-index-luke.com/)
- [PostgreSQL Performance Wiki](https://wiki.postgresql.org/wiki/Performance_Optimization)
- [pgMustard EXPLAIN Analyzer](https://www.pgmustard.com/)
