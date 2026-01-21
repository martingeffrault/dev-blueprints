# PostgreSQL (2025)

> **Last updated**: January 2026
> **Versions covered**: 16, 17
> **Purpose**: The most advanced open-source relational database

---

## Philosophy (2025-2026)

PostgreSQL 17 focuses on **performance optimization, better JSON handling, and improved replication**.

**Key philosophical shifts:**
- **B-tree optimizations** — Multi-value searches in single scan
- **Incremental sort** — Better large dataset handling
- **Logical replication** — Failover slots for zero downtime
- **JSON improvements** — JSONB is production-ready standard
- **Connection pooling** — Essential (PgBouncer, Supavisor)
- **Monitoring first** — pg_stat_statements for query analysis

---

## TL;DR

- Use `timestamp with time zone` (not `without`)
- Use `text` instead of `varchar` (same performance, more flexible)
- Never use `CHAR(n)` — wastes space
- Use `uuid` or `bigserial` for primary keys (avoid UUIDv4 at scale)
- Always add indexes for foreign keys
- Use CTEs for readability (optimized since PG12)
- Monitor with `pg_stat_statements`
- Use connection pooling in production

---

## Best Practices

### Data Types

```sql
-- ✅ GOOD — Proper data types
CREATE TABLE users (
    id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    email           text NOT NULL UNIQUE,
    name            text NOT NULL,
    role            smallint NOT NULL DEFAULT 0,  -- Instead of text enum
    settings        jsonb DEFAULT '{}',
    created_at      timestamptz DEFAULT now(),
    updated_at      timestamptz DEFAULT now()
);

-- ✅ Create indexes
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);
CREATE INDEX idx_users_created_at ON users(created_at DESC);

-- ✅ JSONB index for nested queries
CREATE INDEX idx_users_settings ON users USING gin(settings);
```

### Primary Keys

```sql
-- ✅ OPTION 1: UUID (distributed systems, no sequence coordination)
id uuid PRIMARY KEY DEFAULT gen_random_uuid()

-- ✅ OPTION 2: BIGSERIAL (simpler, better B-tree locality)
id bigserial PRIMARY KEY

-- ✅ OPTION 3: UUIDv7 (ordered, better than v4 for indexes)
-- Requires extension or application-generated
id uuid PRIMARY KEY  -- Generate UUIDv7 in application

-- ⚠️ AVOID at scale: UUIDv4 causes index fragmentation
```

### Timestamps

```sql
-- ✅ ALWAYS use timestamptz
created_at timestamptz DEFAULT now()

-- ❌ DON'T use timestamp without time zone for UTC
created_at timestamp  -- Ambiguous, causes timezone bugs

-- ✅ Auto-update updated_at with trigger
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at();
```

### Indexing Strategies

```sql
-- ✅ B-tree (default) — equality and range queries
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);

-- ✅ Composite index — multi-column queries
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- ✅ Partial index — filtered queries
CREATE INDEX idx_orders_active ON orders(user_id)
    WHERE status = 'active';

-- ✅ GIN index — JSONB and arrays
CREATE INDEX idx_products_tags ON products USING gin(tags);
CREATE INDEX idx_users_settings ON users USING gin(settings jsonb_path_ops);

-- ✅ BRIN index — large sequential datasets (time-series)
CREATE INDEX idx_logs_created_at ON logs USING brin(created_at);

-- ✅ Covering index — include columns to avoid table lookup
CREATE INDEX idx_users_email_covering ON users(email)
    INCLUDE (name, role);
```

### Query Optimization

```sql
-- ✅ Use EXPLAIN ANALYZE to understand query performance
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE user_id = '123' AND status = 'active';

-- ✅ CTEs for readability (not slow since PG12)
WITH recent_orders AS (
    SELECT * FROM orders
    WHERE created_at > now() - interval '30 days'
),
user_totals AS (
    SELECT user_id, SUM(amount) as total
    FROM recent_orders
    GROUP BY user_id
)
SELECT u.name, ut.total
FROM users u
JOIN user_totals ut ON u.id = ut.user_id
ORDER BY ut.total DESC
LIMIT 10;

-- ✅ Use EXISTS instead of IN for subqueries
SELECT * FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.id
    AND o.status = 'completed'
);

-- ✅ PostgreSQL 17: IN clause optimization
-- Now uses single B-tree scan for multiple values
SELECT * FROM products WHERE id IN (1, 2, 3, 4, 5);
```

### JSONB Operations

```sql
-- ✅ Query JSONB fields
SELECT * FROM users
WHERE settings->>'theme' = 'dark';

-- ✅ Check if key exists
SELECT * FROM users
WHERE settings ? 'notifications';

-- ✅ Check nested value
SELECT * FROM users
WHERE settings @> '{"notifications": {"email": true}}';

-- ✅ Update JSONB field
UPDATE users
SET settings = settings || '{"theme": "light"}'
WHERE id = '123';

-- ✅ Remove key from JSONB
UPDATE users
SET settings = settings - 'deprecated_key'
WHERE id = '123';

-- ✅ Create GIN index for JSONB queries
CREATE INDEX idx_settings ON users USING gin(settings jsonb_path_ops);
```

### Transactions and Concurrency

```sql
-- ✅ Use transactions for related operations
BEGIN;
    INSERT INTO orders (user_id, amount) VALUES ('123', 100.00);
    UPDATE users SET balance = balance - 100.00 WHERE id = '123';
COMMIT;

-- ✅ Use FOR UPDATE to lock rows
SELECT * FROM accounts
WHERE id = '123'
FOR UPDATE;

-- ✅ Atomic updates — avoid read-modify-write
UPDATE accounts
SET balance = balance + 100
WHERE id = '123';

-- ❌ DON'T do read-modify-write in application
-- balance = select balance; update balance = balance + 100;
```

### Full-Text Search

```sql
-- ✅ Add search vector column
ALTER TABLE posts ADD COLUMN search_vector tsvector;

-- ✅ Populate search vector
UPDATE posts SET search_vector =
    to_tsvector('english', coalesce(title, '') || ' ' || coalesce(content, ''));

-- ✅ Create GIN index
CREATE INDEX idx_posts_search ON posts USING gin(search_vector);

-- ✅ Search with ranking
SELECT title, ts_rank(search_vector, query) as rank
FROM posts, to_tsquery('english', 'postgresql & performance') query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 10;

-- ✅ Trigger to auto-update search vector
CREATE TRIGGER posts_search_update
    BEFORE INSERT OR UPDATE ON posts
    FOR EACH ROW
    EXECUTE FUNCTION
    tsvector_update_trigger(search_vector, 'pg_catalog.english', title, content);
```

### Configuration Tuning

```sql
-- ✅ Key parameters to tune (postgresql.conf)

-- Memory (adjust based on total RAM)
shared_buffers = '4GB'              -- 25% of RAM
effective_cache_size = '12GB'       -- 50-75% of RAM
work_mem = '64MB'                   -- Per sort/hash operation
maintenance_work_mem = '1GB'        -- For VACUUM, CREATE INDEX

-- Parallelism
max_parallel_workers_per_gather = 4
max_parallel_workers = 8

-- WAL
wal_buffers = '64MB'
checkpoint_completion_target = 0.9

-- Connections
max_connections = 100               -- Use connection pooler!

-- Autovacuum (aggressive for high-write)
autovacuum_vacuum_scale_factor = 0.05
autovacuum_analyze_scale_factor = 0.02
```

### Monitoring

```sql
-- ✅ Enable pg_stat_statements (in postgresql.conf)
-- shared_preload_libraries = 'pg_stat_statements'

-- ✅ Find slow queries
SELECT
    calls,
    round(total_exec_time::numeric, 2) as total_ms,
    round(mean_exec_time::numeric, 2) as mean_ms,
    query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- ✅ Table bloat check
SELECT
    relname as table,
    n_dead_tup as dead_rows,
    n_live_tup as live_rows,
    round(100.0 * n_dead_tup / nullif(n_live_tup + n_dead_tup, 0), 2) as dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY dead_pct DESC;

-- ✅ Index usage
SELECT
    relname as table,
    indexrelname as index,
    idx_scan as scans,
    idx_tup_read as tuples_read
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- ✅ Unused indexes
SELECT
    relname as table,
    indexrelname as index,
    idx_scan as scans
FROM pg_stat_user_indexes
WHERE idx_scan = 0
AND indexrelname NOT LIKE '%_pkey';
```

---

## Anti-Patterns

### ❌ Using CHAR(n)

**Why it's bad**: Pads with spaces, wastes storage, no performance benefit.

```sql
-- ❌ DON'T
status CHAR(10)  -- Padded to 10 chars

-- ✅ DO
status text      -- Or varchar(10) if you need length limit
```

### ❌ Using NOT IN with Subqueries

**Why it's bad**: Returns empty if any NULL in subquery.

```sql
-- ❌ DON'T — Fails silently with NULLs
SELECT * FROM users
WHERE id NOT IN (SELECT user_id FROM banned_users);

-- ✅ DO
SELECT * FROM users u
WHERE NOT EXISTS (
    SELECT 1 FROM banned_users b WHERE b.user_id = u.id
);
```

### ❌ Read-Modify-Write Cycles

**Why it's bad**: Race conditions in concurrent environments.

```sql
-- ❌ DON'T — In application code:
-- balance = SELECT balance FROM accounts WHERE id = 1;
-- UPDATE accounts SET balance = balance + 100 WHERE id = 1;

-- ✅ DO — Atomic update
UPDATE accounts SET balance = balance + 100 WHERE id = 1;

-- ✅ OR use SELECT FOR UPDATE
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
-- Now safely read and modify
UPDATE accounts SET balance = balance + 100 WHERE id = 1;
COMMIT;
```

### ❌ Using timestamp Without Time Zone for UTC

**Why it's bad**: Timezone ambiguity, calculation errors.

```sql
-- ❌ DON'T
created_at timestamp  -- Ambiguous

-- ✅ DO
created_at timestamptz  -- Always includes timezone
```

### ❌ Missing Foreign Key Indexes

**Why it's bad**: Slow joins and cascading deletes.

```sql
-- ❌ DON'T
CREATE TABLE orders (
    user_id uuid REFERENCES users(id)
    -- Missing index on user_id!
);

-- ✅ DO
CREATE TABLE orders (
    user_id uuid REFERENCES users(id)
);
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

### ❌ Entity-Attribute-Value (EAV) Pattern

**Why it's bad**: Impossible to optimize, complex queries, no constraints.

```sql
-- ❌ DON'T
CREATE TABLE attributes (
    entity_id uuid,
    attribute_name text,
    attribute_value text
);

-- ✅ DO — Use JSONB for flexible schema
CREATE TABLE entities (
    id uuid PRIMARY KEY,
    attributes jsonb
);
CREATE INDEX idx_entities_attrs ON entities USING gin(attributes);
```

### ❌ Long-Running Transactions

**Why it's bad**: Blocks autovacuum, causes table bloat.

```sql
-- ❌ DON'T — Keep transactions open for minutes
BEGIN;
SELECT * FROM large_table;
-- ... application does other work ...
-- Transaction stays open, blocking cleanup
COMMIT;

-- ✅ DO — Keep transactions short
BEGIN;
SELECT * FROM large_table;
COMMIT;  -- Immediately
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 16 | 2023 | JSON_ARRAY, JSON_OBJECT, parallel FULL joins |
| 17 | Sept 2024 | B-tree optimization, failover slots, incremental sort |
| 17.x | 2025 | Stability improvements |
| 18 | 2025 | Coming: More performance improvements |

### PostgreSQL 17 Highlights

- **B-tree IN optimization**: Multiple values searched in single scan
- **Correlated subquery optimization**: IN subqueries → JOINs
- **Incremental sort**: Better memory usage for large datasets
- **Failover slots**: Seamless logical replication failover
- **Memory optimization**: Less memory for logical decoding

---

## Quick Reference

| Task | Solution |
|------|----------|
| Create index | `CREATE INDEX name ON table(column)` |
| Partial index | `CREATE INDEX name ON table(col) WHERE condition` |
| GIN index (JSONB) | `CREATE INDEX name ON table USING gin(col)` |
| BRIN index | `CREATE INDEX name ON table USING brin(col)` |
| Explain query | `EXPLAIN ANALYZE query` |
| Vacuum table | `VACUUM ANALYZE table` |
| Reindex | `REINDEX INDEX CONCURRENTLY name` |
| Check bloat | `SELECT * FROM pg_stat_user_tables` |

| Data Type | Use Case |
|-----------|----------|
| `text` | Variable-length strings |
| `integer` / `bigint` | Numbers |
| `numeric` | Exact decimals (money) |
| `boolean` | True/false |
| `uuid` | Unique identifiers |
| `timestamptz` | Timestamps |
| `jsonb` | Semi-structured data |
| `smallint` | Enums (0-32767) |
| `text[]` | Arrays |

---

## Connection Pooling

```bash
# PgBouncer configuration example
[databases]
myapp = host=127.0.0.1 port=5432 dbname=myapp

[pgbouncer]
listen_addr = 127.0.0.1
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction  # Recommended for most apps
max_client_conn = 1000
default_pool_size = 20
```

---

## Resources

- [Official PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [PostgreSQL Wiki - Don't Do This](https://wiki.postgresql.org/wiki/Don't_Do_This)
- [PostgreSQL 17 Release Notes](https://www.postgresql.org/docs/release/17.0/)
- [Top 10 PostgreSQL Best Practices 2025](https://www.instaclustr.com/education/postgresql/top-10-postgresql-best-practices-for-2025/)
- [PostgreSQL Parameter Tuning](https://www.mydbops.com/blog/postgresql-parameter-tuning-best-practices)
- [Five PostgreSQL Anti-Patterns](https://shey.ca/2025/09/12/five-db-anti-patterns.html)
- [10 Postgres Anti-Patterns That Kill Concurrency](https://medium.com/@bhagyarana80/10-postgres-anti-patterns-that-kill-concurrency-df368e30031d)
