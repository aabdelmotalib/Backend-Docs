# Module 5: Indexes and Query Performance

## The Analogy: Library Card Catalog

Without an Index:
- Looking for "Advanced Python" — read every single card in the drawer. 1000 cards → 500 average reads.

With an Index:
- Catalog is organized alphabetically. Open to "A". Find book in 10 reads.

Indexes work the same way:

**Without index** — 100,000 users, search by email: PostgreSQL reads all 100,000 rows.
**With index** — PostgreSQL reads ~20 rows using the index.

## Index Types

### Primary Key Index (Automatic)

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,  -- Automatically indexed
    email VARCHAR(255)
);
```

Every table gets an index on the primary key automatically.

### Simple Index

```sql
CREATE INDEX idx_users_email ON users(email);
```

Now queries on email are fast:

```sql
SELECT * FROM users WHERE email = 'alice@example.com';  -- Uses index!
```

### Unique Index

```sql
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);
```

Combinations:

- Prevents duplicates (like UNIQUE constraint)
- Acts as an index (queries are fast)

### Composite Index

Multiple columns:

```sql
CREATE INDEX idx_jobs_sub_status ON jobs(subscription_id, status);
```

Fast for queries like:

```sql
SELECT * FROM jobs 
WHERE subscription_id = 5 AND status = 'processing';  -- Uses index!
```

Order matters. This index helps when filtering by `subscription_id` first.

### Partial Index

Only index rows matching a condition:

```sql
CREATE INDEX idx_active_subs ON subscriptions(user_id) 
WHERE active = TRUE;
```

Fast for:

```sql
SELECT * FROM subscriptions WHERE user_id = 1 AND active = TRUE;
```

Saves space (doesn't index inactive rows).

## EXPLAIN ANALYZE: Measuring Query Speed

Before optimizing, measure.

```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'alice@example.com';
```

Output (without index):

```
Seq Scan on users  (cost=0.00..35.50 rows=1 width=500)
  Filter: (email = 'alice@example.com')
  Actual Time: 5.234..15.234 rows=1 loops=1
Planning Time: 0.031 ms
Execution Time: 15.265 ms
```

Translation: This **sequential scan** (no index) took 15.265 ms.

After creating index:

```sql
CREATE INDEX idx_users_email ON users(email);
```

```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'alice@example.com';
```

Output (with index):

```
Index Scan using idx_users_email on users  (cost=0.29..8.30 rows=1 width=500)
  Index Cond: (email = 'alice@example.com')
  Actual Time: 0.023..0.031 rows=1 loops=1
Planning Time: 0.043 ms
Execution Time: 0.062 ms
```

Translation: This **index scan** took only 0.062 ms (250x faster!).

## When to Add Indexes

Add indexes on columns you:

1. **Filter by** — `WHERE email = ...`
2. **Join on** — `ON users.id = subscriptions.user_id`
3. **Sort by** — `ORDER BY created_at`
4. **Query frequently** — High-traffic queries

### Example: PDF Processing Jobs

```sql
CREATE TABLE jobs (
    id SERIAL PRIMARY KEY,
    subscription_id INTEGER NOT NULL REFERENCES subscriptions(id),
    status VARCHAR(50),
    created_at TIMESTAMP,
    completed_at TIMESTAMP
);

-- Add indexes
CREATE INDEX idx_jobs_sub_status ON jobs(subscription_id, status);
CREATE INDEX idx_jobs_created_at ON jobs(created_at DESC);
CREATE INDEX idx_jobs_completed ON jobs(completed_at) WHERE completed_at IS NOT NULL;
```

Speeds up queries:

```sql
-- Finds jobs in specific subscription with specific status
SELECT * FROM jobs WHERE subscription_id = 5 AND status = 'processing';

-- Finds recent jobs
SELECT * FROM jobs ORDER BY created_at DESC LIMIT 10;

-- Finds completed jobs
SELECT * FROM jobs WHERE completed_at IS NOT NULL ORDER BY completed_at DESC;
```

## Index Downsides

Indexes are not free:

1. **Storage** — Indexes use disk space (usually small)
2. **Write Performance** — INSERT/UPDATE/DELETE must update indexes (slower)
3. **Maintenance** — PostgreSQL must keep indexes organized

Rules of thumb:

- **Read-heavy workload** → Many indexes (good trade-off)
- **Write-heavy workload** → Few indexes (writes are expensive)

## Finding Missing Indexes

PostgreSQL tracks which indexes are used:

```sql
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;
```

Indexes with `idx_scan = 0` are never used. Consider dropping them.

```sql
-- Drop unused index
DROP INDEX idx_users_unused;
```

## B-Tree Indexes

PostgreSQL uses **B-Tree** indexes by default. They:

- Support range queries: `WHERE age > 18`
- Support sorting: `ORDER BY age`
- Support equality: `WHERE email = '...'`

For most queries, B-Tree is sufficient.

## Hands-On Lab

### Lab 5.1: Measure Without Index

```sql
-- Create 10,000 test users
CREATE TABLE test_users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255),
    name VARCHAR(100)
);

INSERT INTO test_users (email, name)
SELECT 'user' || i || '@example.com', 'User ' || i
FROM generate_series(1, 10000) i;

-- Query without index (SLOW)
EXPLAIN ANALYZE
SELECT * FROM test_users WHERE email = 'user9000@example.com';
```

Note execution time.

### Lab 5.2: Add Index and Remeasure

```sql
CREATE INDEX idx_test_email ON test_users(email);

-- Same query with index (FAST)
EXPLAIN ANALYZE
SELECT * FROM test_users WHERE email = 'user9000@example.com';
```

Compare times from Lab 1. Should be significantly faster.

### Lab 5.3: Composite Index

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER,
    status VARCHAR(50),
    amount DECIMAL(10, 2)
);

INSERT INTO orders (user_id, status, amount)
SELECT 
    (RANDOM() * 1000)::INT,
    CASE (RANDOM() * 3)::INT
        WHEN 0 THEN 'pending'
        WHEN 1 THEN 'completed'
        ELSE 'failed'
    END,
    RANDOM() * 1000
FROM generate_series(1, 100000) i;

-- Query
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 500 AND status = 'completed';
```

Without index: slow. Add index:

```sql
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

Requery. Now it's fast.

## Cheat Sheet: Indexes

### Create Index

```sql
CREATE INDEX idx_users_email ON users(email);
CREATE UNIQUE INDEX idx_email_unique ON users(email);
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at DESC);
CREATE INDEX idx_active_subs ON subs(user_id) WHERE active = TRUE;
```

### Measure Performance

```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'alice@example.com';
```

### Drop Index

```sql
DROP INDEX idx_users_email;
```

### List Indexes

```sql
SELECT tablename, indexname FROM pg_indexes WHERE tablename = 'users';
```

### Find Unused Indexes

```sql
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;
```

## Key Takeaways

- **Indexes speed up reads at cost of writes** — good for read-heavy workloads
- **EXPLAIN ANALYZE measures performance** — use it before and after adding indexes
- **Index on columns you filter/join/sort by** — email, user_id, created_at
- **Composite indexes** — (column1, column2) fast for combined queries
- **B-Tree is the default** — works for most queries
- **Monitor unused indexes** — drop them to save space
- **Partial indexes** — reduce size by filtering rows

Module 6 teaches Transactions — safe handling of simultaneous changes.
