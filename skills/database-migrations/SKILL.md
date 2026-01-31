# Database Migrations

Procedural guide for safe database migrations with zero-downtime strategies.

## When to Use

- Adding, modifying, or removing columns/tables
- Creating or dropping indexes
- Changing constraints or data types
- Deploying schema changes to production
- Rolling back failed migrations

## Migration Safety Checklist

Before running any migration:
- [ ] Tested on staging with production-like data
- [ ] Reviewed for locking behavior
- [ ] Has rollback plan
- [ ] Estimated execution time (EXPLAIN)
- [ ] Notified team of maintenance window (if needed)

## Safe Operations (No/Minimal Lock)

### ✅ Always Safe
```sql
-- Add nullable column
ALTER TABLE users ADD COLUMN bio TEXT;

-- Add column with default (PG 11+, instant)
ALTER TABLE users ADD COLUMN verified BOOLEAN DEFAULT false;

-- Create index concurrently
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);

-- Drop index concurrently
DROP INDEX CONCURRENTLY idx_users_email;

-- Add new table
CREATE TABLE new_feature (...);

-- Drop unused table
DROP TABLE old_feature;
```

## Dangerous Operations (Require Care)

### ⚠️ Adding NOT NULL Constraint

**Problem:** `ALTER TABLE ... SET NOT NULL` scans entire table.

**Safe Pattern:**
```sql
-- Step 1: Add nullable column (instant)
ALTER TABLE users ADD COLUMN status TEXT;

-- Step 2: Backfill data (can be batched)
UPDATE users SET status = 'active' WHERE status IS NULL;

-- Step 3: Add NOT VALID constraint (instant, no scan)
ALTER TABLE users ADD CONSTRAINT users_status_not_null 
  CHECK (status IS NOT NULL) NOT VALID;

-- Step 4: Validate in background (no exclusive lock)
ALTER TABLE users VALIDATE CONSTRAINT users_status_not_null;

-- Step 5 (optional): Convert to column NOT NULL
-- Only if you need the explicit column constraint
ALTER TABLE users ALTER COLUMN status SET NOT NULL;
ALTER TABLE users DROP CONSTRAINT users_status_not_null;
```

### ⚠️ Changing Column Type

**Problem:** May rewrite entire table.

**Safe Pattern:**
```sql
-- Step 1: Add new column
ALTER TABLE products ADD COLUMN price_numeric NUMERIC(10,2);

-- Step 2: Backfill (batched)
UPDATE products SET price_numeric = price::NUMERIC(10,2) 
WHERE id BETWEEN 1 AND 10000;
-- Repeat for batches...

-- Step 3: Update application to write both columns

-- Step 4: Validate data
SELECT COUNT(*) FROM products WHERE price_numeric IS NULL;

-- Step 5: Switch application to read new column

-- Step 6: Drop old column (in separate migration)
ALTER TABLE products DROP COLUMN price;

-- Step 7: Rename (optional)
ALTER TABLE products RENAME COLUMN price_numeric TO price;
```

### ⚠️ Renaming Column

**Problem:** Breaks application code instantly.

**Safe Pattern:**
```sql
-- Step 1: Add new column
ALTER TABLE users ADD COLUMN full_name TEXT;

-- Step 2: Backfill
UPDATE users SET full_name = name WHERE full_name IS NULL;

-- Step 3: Create trigger to sync both columns
CREATE OR REPLACE FUNCTION sync_user_name() RETURNS TRIGGER AS $$
BEGIN
  IF NEW.name IS DISTINCT FROM OLD.name THEN
    NEW.full_name := NEW.name;
  ELSIF NEW.full_name IS DISTINCT FROM OLD.full_name THEN
    NEW.name := NEW.full_name;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER sync_name_trigger
  BEFORE UPDATE ON users
  FOR EACH ROW EXECUTE FUNCTION sync_user_name();

-- Step 4: Update application to use new column

-- Step 5: Remove trigger and old column
DROP TRIGGER sync_name_trigger ON users;
DROP FUNCTION sync_user_name();
ALTER TABLE users DROP COLUMN name;
```

### ⚠️ Adding Foreign Key

**Problem:** Validates all existing rows.

**Safe Pattern:**
```sql
-- Step 1: Add constraint as NOT VALID (instant)
ALTER TABLE orders 
  ADD CONSTRAINT fk_orders_user 
  FOREIGN KEY (user_id) REFERENCES users(id) NOT VALID;

-- Step 2: Validate in background (no exclusive lock)
ALTER TABLE orders VALIDATE CONSTRAINT fk_orders_user;
```

### ⚠️ Dropping Column

**Problem:** May cause application errors if code still references it.

**Safe Pattern:**
```sql
-- Step 1: Update application to stop reading/writing column

-- Step 2: Wait for all old code to be replaced (1+ deploys)

-- Step 3: Drop column
ALTER TABLE users DROP COLUMN old_field;
```

## Index Strategies

### Create Index Concurrently
```sql
-- Non-blocking (takes longer but no lock)
CREATE INDEX CONCURRENTLY idx_orders_status ON orders(status);

-- Check if index creation is stuck
SELECT * FROM pg_stat_progress_create_index;

-- If failed, check for INVALID index
SELECT indexname, indexdef FROM pg_indexes 
WHERE schemaname = 'public' AND indexname LIKE '%_ccnew%';

-- Drop invalid index and retry
DROP INDEX CONCURRENTLY idx_orders_status_ccnew;
```

### Index for Specific Queries
```sql
-- Partial index (smaller, faster)
CREATE INDEX CONCURRENTLY idx_orders_pending 
ON orders(created_at) 
WHERE status = 'pending';

-- Covering index (index-only scan)
CREATE INDEX CONCURRENTLY idx_orders_user_covering
ON orders(user_id)
INCLUDE (status, total);
```

## Batched Updates

For large backfills, batch to avoid long transactions:

```sql
-- Batch update with progress tracking
DO $$
DECLARE
  batch_size INT := 10000;
  rows_updated INT;
  total_updated INT := 0;
BEGIN
  LOOP
    UPDATE users
    SET new_field = 'default'
    WHERE id IN (
      SELECT id FROM users 
      WHERE new_field IS NULL 
      LIMIT batch_size
    );
    
    GET DIAGNOSTICS rows_updated = ROW_COUNT;
    total_updated := total_updated + rows_updated;
    
    RAISE NOTICE 'Updated % rows, total: %', rows_updated, total_updated;
    
    EXIT WHEN rows_updated = 0;
    
    -- Small pause to reduce load
    PERFORM pg_sleep(0.1);
  END LOOP;
END $$;
```

## Rollback Strategies

### Additive Changes (Easy Rollback)
```sql
-- Migration: Add column
ALTER TABLE users ADD COLUMN preferences JSONB;

-- Rollback: Drop column
ALTER TABLE users DROP COLUMN preferences;
```

### Destructive Changes (Plan Ahead)
```sql
-- Before dropping: Create backup table
CREATE TABLE users_backup_20240115 AS SELECT * FROM users;

-- Or: Use soft delete
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMP;
-- Instead of DELETE, SET deleted_at = NOW()

-- Rollback: Restore from backup
INSERT INTO users SELECT * FROM users_backup_20240115 
ON CONFLICT (id) DO NOTHING;
```

## Migration Tools

### golang-migrate
```bash
# Create migration
migrate create -ext sql -dir migrations -seq add_user_status

# Run migrations
migrate -path migrations -database "postgres://..." up

# Rollback last migration
migrate -path migrations -database "postgres://..." down 1

# Force version (after failed migration)
migrate -path migrations -database "postgres://..." force 20240115
```

### Prisma
```bash
# Create migration
npx prisma migrate dev --name add_user_status

# Apply migrations (production)
npx prisma migrate deploy

# Reset database (dev only!)
npx prisma migrate reset
```

### Drizzle
```bash
# Generate migration
npx drizzle-kit generate:pg

# Push changes (dev)
npx drizzle-kit push:pg

# Apply migrations
npx drizzle-kit migrate
```

## Zero-Downtime Deployment Pattern

1. **Deploy new code** that handles both old and new schema
2. **Run forward migration** (add columns, create indexes)
3. **Deploy code** that uses new schema
4. **Run cleanup migration** (drop old columns)

```
v1 code  ──▶  v1.5 code (handles both)  ──▶  v2 code
              ▲
              │ Migration runs here
              │ (while v1.5 is live)
```

## Monitoring During Migration

```sql
-- Check for locks
SELECT 
  pid,
  age(clock_timestamp(), query_start),
  usename,
  query,
  state
FROM pg_stat_activity
WHERE state != 'idle'
  AND query NOT ILIKE '%pg_stat_activity%'
ORDER BY query_start;

-- Check lock waits
SELECT 
  blocked_locks.pid AS blocked_pid,
  blocking_locks.pid AS blocking_pid,
  blocked_activity.query AS blocked_query,
  blocking_activity.query AS blocking_query
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_locks blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
JOIN pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;

-- Kill stuck query if needed
SELECT pg_terminate_backend(pid);
```

## Common Mistakes

### ❌ Running migrations without testing
Always test on staging with production-like data volume.

### ❌ No rollback plan
Every migration should have a documented rollback procedure.

### ❌ Mixing schema and data changes
Keep schema changes and data backfills in separate migrations.

### ❌ Long-running transactions
Batch large updates; don't run hours-long transactions.

### ❌ Forgetting `CONCURRENTLY`
Always use `CREATE INDEX CONCURRENTLY` in production.

## References

- [PostgreSQL ALTER TABLE](https://www.postgresql.org/docs/current/sql-altertable.html)
- [Zero-Downtime Postgres Migrations](https://www.braintreepayments.com/blog/safe-operations-for-high-volume-postgresql/)
- [Strong Migrations Gem Docs](https://github.com/ankane/strong_migrations)
