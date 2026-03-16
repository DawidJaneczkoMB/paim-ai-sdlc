# SQL & Database Engineering — Full Reference

## 1. Database Schema Architecture Design

### Primary Key Strategy

| Strategy | When to Use | Trade-offs |
|---|---|---|
| `UUID v4` / `UUID v7` | Distributed systems, microservices, public-facing IDs | Larger storage, random UUIDs cause index fragmentation (prefer UUID v7 for ordered inserts) |
| `ULID` | When you need sortable unique IDs with good index locality | Not natively supported in all databases |
| `BIGSERIAL` / `IDENTITY` | Single-database monoliths, internal IDs | Sequential, predictable (security concern if exposed), not portable across DB instances |

**Rule:** Never expose auto-increment IDs in public APIs. Use UUIDs or ULIDs for external identifiers.

### Enum Handling

Prefer CHECK constraints over native ENUM types for portability and ease of migration:

```sql
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    status VARCHAR(20) NOT NULL DEFAULT 'pending'
        CONSTRAINT chk_orders_status CHECK (status IN ('pending', 'confirmed', 'shipped', 'delivered', 'cancelled')),
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
```

Adding a new enum value is a simple `ALTER TABLE ... DROP CONSTRAINT` + `ADD CONSTRAINT`, no table rewrite required.

### Schema Design Checklist

```
Schema design checklist:
- [ ] Every table has a primary key (UUID or BIGSERIAL)
- [ ] Every table has created_at and updated_at columns
- [ ] All foreign keys have corresponding indexes
- [ ] All columns have appropriate NOT NULL constraints
- [ ] Unique constraints are defined where business rules require uniqueness
- [ ] CHECK constraints enforce valid value ranges and enums
- [ ] Naming follows snake_case conventions consistently
- [ ] Data types are chosen precisely (no FLOAT for money, no TEXT for emails)
- [ ] Soft delete (deleted_at) is used only where business rules require it
- [ ] No business logic is embedded in database triggers or stored procedures (keep logic in the application layer)
```

---

## 2. Normalisation Strategies

### Normal Forms Reference

| Normal Form | Rule | Violation Example | Fix |
|---|---|---|---|
| **1NF** | Every column holds atomic values. No repeating groups. | `tags: "php,sql,go"` in a single column | Create a separate `tags` table with a junction table |
| **2NF** | 1NF + every non-key column depends on the **entire** primary key | `order_items(order_id, product_id, product_name)` — `product_name` depends only on `product_id` | Move `product_name` to the `products` table |
| **3NF** | 2NF + no transitive dependencies | `employees(id, department_id, department_name)` — `department_name` depends on `department_id` | Move `department_name` to a `departments` table |
| **BCNF** | 3NF + every determinant is a candidate key | Rare in practice | Decompose the table so every determinant is a key |

**Target 3NF by default.** Use 2NF only as intermediate step when refactoring legacy. Use BCNF when composite primary keys create functional dependency issues.

### Practical Normalisation Example

**Unnormalised (0NF):**

```
orders:
| order_id | customer_name | customer_email      | items                          |
|----------|---------------|---------------------|--------------------------------|
| 1        | John Doe      | john@example.com    | Widget x2, Gadget x1           |
```

**1NF — Atomic values, no repeating groups:**

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_name VARCHAR(100) NOT NULL,
    customer_email VARCHAR(255) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE TABLE order_items (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id),
    product_name VARCHAR(100) NOT NULL,
    quantity INT NOT NULL CHECK (quantity > 0)
);
```

**2NF — Remove partial dependencies:**

```sql
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    price NUMERIC(19,4) NOT NULL
);

CREATE TABLE order_items (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id),
    product_id BIGINT NOT NULL REFERENCES products(id),
    quantity INT NOT NULL CHECK (quantity > 0),
    unit_price NUMERIC(19,4) NOT NULL  -- snapshot of price at order time
);
```

**3NF — Remove transitive dependencies:**

```sql
CREATE TABLE customers (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE
);

CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL REFERENCES customers(id),
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
```

### Strategic Denormalisation

Denormalise **only** when you have measured performance data proving joins are the bottleneck:

| Pattern | When to Use | Example |
|---|---|---|
| **Materialised/cached columns** | Frequently read expensive aggregate | `orders.total_amount` from `order_items` |
| **Snapshot columns** | Values preserved at a point in time | `order_items.unit_price` (price at purchase) |
| **Read-optimised views** | Reporting/analytics spanning many tables | Materialised views refreshed periodically |
| **Search/filter columns** | Columns from related tables used heavily in WHERE | `orders.customer_country` duplicated for filtering |

**Rules:**
- Always document **why** the denormalisation exists.
- Ensure denormalised data is kept in sync (application-level updates, triggers as last resort).
- Treat denormalised data as a **cache** — normalised source remains the source of truth.

---

## 3. Relationships & Foreign Keys

### Relationship Types

**One-to-Many (most common):**

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL REFERENCES customers(id) ON DELETE RESTRICT,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_orders_customer_id ON orders(customer_id);
```

**Many-to-Many (junction table):**

```sql
CREATE TABLE products_tags (
    product_id BIGINT NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    tag_id BIGINT NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (product_id, tag_id)
);

CREATE INDEX idx_products_tags_tag_id ON products_tags(tag_id);
```

**One-to-One:**

```sql
CREATE TABLE user_profiles (
    user_id BIGINT PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    bio TEXT,
    avatar_url VARCHAR(500)
);
```

### ON DELETE / ON UPDATE Actions

| Action | Meaning | When to Use |
|---|---|---|
| `RESTRICT` (default) | Prevent deletion if referenced rows exist | Default for most business entities |
| `CASCADE` | Delete/update child rows when parent is deleted/updated | Junction tables, tightly coupled children |
| `SET NULL` | Set FK to NULL when parent is deleted | Optional relationships where child exists independently |
| `SET DEFAULT` | Set FK to default value | Rare — when fallback reference makes business sense |
| `NO ACTION` | Same as RESTRICT but checked at end of transaction | When deferred constraint checking is needed |

**Rule:** Default to `RESTRICT`. Use `CASCADE` only on junction tables and tightly coupled children. Always explicitly specify the action.

### Self-Referencing Relationships

```sql
CREATE TABLE categories (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    parent_id BIGINT REFERENCES categories(id) ON DELETE SET NULL
);

CREATE INDEX idx_categories_parent_id ON categories(parent_id);
```

For deep hierarchies:
- **Adjacency list** — simple, good for shallow trees
- **Materialised path** — `path VARCHAR(500)` storing `/1/5/12/` — read-heavy trees
- **Nested sets** — complex to maintain but fast for subtree queries
- **Closure table** — separate table storing all ancestor-descendant pairs — balanced read/write

---

## 4. Indexes

### Index Types

| Type | Syntax (PostgreSQL) | Use Case |
|---|---|---|
| **B-tree** (default) | `CREATE INDEX idx ON table(column)` | Equality and range queries (`=`, `<`, `>`, `BETWEEN`, `ORDER BY`) |
| **Hash** | `CREATE INDEX idx ON table USING hash(column)` | Equality-only lookups (rare) |
| **GIN** | `CREATE INDEX idx ON table USING gin(column)` | Full-text search, JSONB containment, array operations |
| **GiST** | `CREATE INDEX idx ON table USING gist(column)` | Geometric/spatial data, range types, nearest-neighbor |
| **BRIN** | `CREATE INDEX idx ON table USING brin(column)` | Very large tables with naturally ordered data |

### Index Strategy Rules

**Always index:** Foreign key columns, columns in `WHERE`, columns in `ORDER BY` on large tables, columns in `JOIN` conditions.

**Composite indexes:** For queries filtering on multiple columns together, or covering indexes.

**Avoid over-indexing:** Each index adds `INSERT`/`UPDATE`/`DELETE` overhead. Review and remove unused indexes periodically.

### Partial Indexes

```sql
CREATE INDEX idx_orders_active ON orders(customer_id, created_at)
    WHERE deleted_at IS NULL;
```

### Unique Indexes

```sql
CREATE UNIQUE INDEX uq_users_email_active ON users(email)
    WHERE deleted_at IS NULL;
```

### Index Monitoring

```sql
-- PostgreSQL: find unused indexes
SELECT
    schemaname, relname AS table_name, indexrelname AS index_name,
    idx_scan AS times_used, pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

### Non-Blocking Index Operations (CONCURRENTLY)

**Always use `CONCURRENTLY`** when creating or dropping indexes on tables serving live traffic:

```sql
CREATE INDEX CONCURRENTLY idx_orders_customer_id ON orders(customer_id);
DROP INDEX CONCURRENTLY idx_orders_customer_id;
```

| Aspect | Detail |
|---|---|
| **Database support** | PostgreSQL-specific. MySQL: `ALGORITHM=INPLACE, LOCK=NONE`; SQL Server: `WITH (ONLINE = ON)`; Oracle: `ONLINE` |
| **Transaction blocks** | Cannot run inside `BEGIN`/`COMMIT` |
| **Migration frameworks** | Most wrap in transactions by default — disable for concurrent ops |
| **Failed builds** | Leaves invalid index — check with `\d table_name` and drop before retrying |

**Skip `CONCURRENTLY` for:** Initial schema setup, empty tables, maintenance windows, test environments.

---

## 5. Writing Performant SQL Queries

### JOIN Optimisation

| Join Type | Returns | Use When |
|---|---|---|
| `INNER JOIN` | Only matching rows from both tables | Data must exist in both tables |
| `LEFT JOIN` | All left + matches from right (NULL if no match) | Need all records from primary table |
| `RIGHT JOIN` | All right + matches from left | Rarely used — rewrite as LEFT JOIN |
| `CROSS JOIN` | Cartesian product | Generating combinations |
| `LATERAL JOIN` | Correlated subquery for each row | Top-N per group, complex per-row calculations |

```sql
-- GOOD: Join on indexed columns, filter early
SELECT o.id, o.created_at, c.name
FROM orders o
INNER JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'active'
  AND o.created_at >= '2025-01-01'
ORDER BY o.created_at DESC
LIMIT 50;

-- BAD: Joining on expressions, filtering late
SELECT *
FROM orders o
INNER JOIN customers c ON LOWER(c.email) = LOWER(o.customer_email)
WHERE YEAR(o.created_at) = 2025;
```

### Pagination Strategies

**Offset-based** (simple but degrades on large offsets):

```sql
SELECT id, name, email
FROM users
WHERE deleted_at IS NULL
ORDER BY created_at DESC
LIMIT 20 OFFSET 1000;
```

**Keyset/cursor-based** (performant at any depth):

```sql
-- First page
SELECT id, name, email, created_at
FROM users
WHERE deleted_at IS NULL
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- Next page (use last row's values as cursor)
SELECT id, name, email, created_at
FROM users
WHERE deleted_at IS NULL
  AND (created_at, id) < ('2025-02-01T10:00:00Z', '550e8400-e29b-41d4-a716-446655440000')
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

**Rule:** Offset for admin UIs with moderate data (< 100k rows). Keyset for APIs, infinite scrolling, large datasets.

### Common Table Expressions (CTEs)

```sql
WITH active_customers AS (
    SELECT id, name
    FROM customers
    WHERE status = 'active'
      AND deleted_at IS NULL
),
recent_orders AS (
    SELECT customer_id, COUNT(*) AS order_count
    FROM orders
    WHERE created_at >= NOW() - INTERVAL '30 days'
    GROUP BY customer_id
)
SELECT ac.id, ac.name, COALESCE(ro.order_count, 0) AS recent_orders
FROM active_customers ac
LEFT JOIN recent_orders ro ON ro.customer_id = ac.id
ORDER BY recent_orders DESC;
```

### Window Functions

```sql
SELECT
    customer_id,
    id AS order_id,
    total,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY total DESC) AS rank,
    SUM(total) OVER (PARTITION BY customer_id) AS customer_total
FROM orders
WHERE status = 'completed';
```

### Batch Operations

```sql
DO $$
DECLARE
    rows_deleted INT;
BEGIN
    LOOP
        DELETE FROM logs
        WHERE id IN (
            SELECT id FROM logs
            WHERE created_at < '2024-01-01'
            LIMIT 5000
        );
        GET DIAGNOSTICS rows_deleted = ROW_COUNT;
        EXIT WHEN rows_deleted = 0;
        COMMIT;
    END LOOP;
END $$;
```

### NULL Handling

```sql
-- WRONG: = NULL never matches
SELECT * FROM users WHERE deleted_at = NULL;

-- CORRECT: Use IS NULL / IS NOT NULL
SELECT * FROM users WHERE deleted_at IS NULL;

-- Use COALESCE for default values
SELECT COALESCE(u.nickname, u.first_name, 'Anonymous') AS display_name
FROM users u;

-- NULL-safe comparison
-- PostgreSQL:
SELECT * FROM t1 JOIN t2 ON t1.val IS NOT DISTINCT FROM t2.val;
-- MySQL:
SELECT * FROM t1 JOIN t2 ON t1.val <=> t2.val;
```

### UPSERT Patterns

```sql
-- PostgreSQL: INSERT ... ON CONFLICT
INSERT INTO products (sku, name, price, updated_at)
VALUES ('WIDGET-001', 'Widget', 19.99, NOW())
ON CONFLICT (sku)
DO UPDATE SET
    name = EXCLUDED.name,
    price = EXCLUDED.price,
    updated_at = NOW();

-- MySQL: INSERT ... ON DUPLICATE KEY UPDATE
INSERT INTO products (sku, name, price, updated_at)
VALUES ('WIDGET-001', 'Widget', 19.99, NOW())
ON DUPLICATE KEY UPDATE
    name = VALUES(name),
    price = VALUES(price),
    updated_at = NOW();
```

### Formatting Standards

```sql
SELECT
    u.id,
    u.email,
    u.first_name,
    u.last_name,
    COUNT(o.id) AS order_count,
    COALESCE(SUM(o.total), 0) AS lifetime_value
FROM users u
LEFT JOIN orders o ON o.customer_id = u.id
    AND o.status = 'completed'
WHERE u.deleted_at IS NULL
  AND u.created_at >= '2025-01-01'
GROUP BY u.id, u.email, u.first_name, u.last_name
HAVING COUNT(o.id) > 0
ORDER BY lifetime_value DESC
LIMIT 50;
```

**Rules:**
- Keywords in UPPERCASE: `SELECT`, `FROM`, `WHERE`, `JOIN`, `ORDER BY`, `GROUP BY`
- One column per line in `SELECT` for queries with 3+ columns
- `AND`/`OR` at beginning of line, indented
- Table aliases: short but meaningful (`u` for `users`, `o` for `orders`)
- Always terminate statements with `;`

### Security — Preventing SQL Injection

**Never concatenate user input into SQL strings.** Always use parameterised queries:

```sql
-- PostgreSQL / Node.js
SELECT * FROM users WHERE email = $1;

-- MySQL / PHP
SELECT * FROM users WHERE email = ?;
```

**ORM equivalents (all safe by default):**

```typescript
// TypeORM
const user = await userRepository.findOne({ where: { email: userInput } });

// Prisma
const user = await prisma.user.findUnique({ where: { email: userInput } });
```

**DANGER: Raw queries with string concatenation:**

```typescript
// DANGEROUS
const users = await dataSource.query(`SELECT * FROM users WHERE email = '${userInput}'`);

// SAFE
const users = await dataSource.query('SELECT * FROM users WHERE email = $1', [userInput]);
```

---

## 6. Transactions

### ACID Properties

| Property | Meaning | Practical Impact |
|---|---|---|
| **Atomicity** | All operations succeed or all are rolled back | Partial updates never reach the database |
| **Consistency** | Transaction moves DB from one valid state to another | Constraints enforced at commit time |
| **Isolation** | Concurrent transactions don't interfere | Determined by isolation level |
| **Durability** | Committed data survives system failures | Data persisted to disk on commit |

### Transaction Isolation Levels

| Level | Dirty Reads | Non-Repeatable Reads | Phantom Reads | Use Case |
|---|---|---|---|---|
| `READ UNCOMMITTED` | Yes | Yes | Yes | Almost never — only for dirty analytics |
| `READ COMMITTED` | No | Yes | Yes | **Default for PostgreSQL.** Most OLTP workloads |
| `REPEATABLE READ` | No | No | Yes* | Financial calculations, inventory checks |
| `SERIALIZABLE` | No | No | No | Critical financial operations, booking systems |

*PostgreSQL's `REPEATABLE READ` also prevents phantom reads (snapshot isolation).

**Rule:** Use `READ COMMITTED` by default. Escalate only for specific operations and handle serialisation failures with retry logic.

### Transaction Best Practices

```sql
-- Short, focused transaction
BEGIN;
    UPDATE accounts SET balance = balance - 100.00 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100.00 WHERE id = 2;
    INSERT INTO transactions (from_id, to_id, amount) VALUES (1, 2, 100.00);
COMMIT;
```

**Rules:**
- Keep transactions as short as possible.
- Never perform external I/O (HTTP calls, file ops) inside a transaction.
- Always handle failures: catch exceptions, rollback, optionally retry.
- Use `SAVEPOINT` for partial rollbacks within larger transactions.
- Acquire locks in consistent order to prevent deadlocks (e.g., ascending ID).

### Savepoints

```sql
BEGIN;
    INSERT INTO orders (customer_id, total) VALUES (1, 250.00);
    SAVEPOINT before_items;

    INSERT INTO order_items (order_id, product_id, quantity) VALUES (1, 99, 1);
    -- If this fails:
    ROLLBACK TO SAVEPOINT before_items;

COMMIT;
```

---

## 7. Locking Mechanics

### Lock Types

| Lock Type | SQL | Scope | Behaviour |
|---|---|---|---|
| **Row-level shared** | `SELECT ... FOR SHARE` | Row | Others can read but not modify |
| **Row-level exclusive** | `SELECT ... FOR UPDATE` | Row | Others cannot read (with FOR UPDATE) or modify |
| **Table-level** | `LOCK TABLE ... IN <mode> MODE` | Table | Entire table — use sparingly |
| **Advisory locks** | `pg_advisory_lock(key)` | App-defined | Application-level coordination |

### Optimistic Locking

Assumes conflicts are rare. Checks at write time via version column:

```sql
ALTER TABLE products ADD COLUMN version INT NOT NULL DEFAULT 1;

-- Read
SELECT id, name, price, stock, version FROM products WHERE id = 42;

-- Update with version check
UPDATE products
SET stock = stock - 1, version = version + 1, updated_at = NOW()
WHERE id = 42 AND version = 3;
-- 0 rows affected → conflict → retry or error
```

**When to use:** Low contention, read-heavy, user-facing forms, distributed systems.

**ORM support:**

| ORM | Implementation |
|---|---|
| **TypeORM** | `@VersionColumn()` decorator |
| **Prisma** | Manual via `version` field and conditional update |
| **Doctrine** | `@ORM\Version` annotation |
| **Eloquent** | Manual or `laravel-optimistic-locking` package |
| **Entity Framework** | `[Timestamp]` attribute or `IsRowVersion()` |
| **Hibernate** | `@Version` annotation |
| **GORM** | `gorm:"column:version"` tag |

### Pessimistic Locking

Assumes conflicts are likely. Acquires locks at read time:

```sql
BEGIN;
    SELECT * FROM inventory WHERE product_id = 42 FOR UPDATE;
    UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 42;
COMMIT;
```

**Variants:**

```sql
-- SKIP LOCKED — skip locked rows (job queues)
SELECT id, payload
FROM job_queue
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;

-- NOWAIT — fail immediately if locked
SELECT * FROM inventory WHERE product_id = 42 FOR UPDATE NOWAIT;
```

**When to use:** High contention, critical sections (inventory, bookings), short transactions, job queues (`SKIP LOCKED`).

### Deadlock Prevention

1. **Consistent lock ordering** — always by ascending primary key.
2. **Short transactions** — minimise lock hold time.
3. **Lock timeout** — set to avoid indefinite blocking.
4. **Detect and retry** — handle deadlock exceptions.

```sql
-- PostgreSQL
SET lock_timeout = '5s';

-- MySQL
SET innodb_lock_wait_timeout = 5;
```

---

## 8. Query Debugging with EXPLAIN

### EXPLAIN Basics

```sql
-- Show query plan (does not execute)
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;

-- Show plan AND execute (actual timings)
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;

-- Full diagnostic (PostgreSQL)
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.id, c.name
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'active'
ORDER BY o.created_at DESC
LIMIT 20;
```

### Common Performance Problems and Fixes

| EXPLAIN Symptom | Likely Cause | Fix |
|---|---|---|
| `Seq Scan` on large table | Missing index | Add appropriate index |
| `Seq Scan` despite index | Function on indexed column | Rewrite query or create functional index |
| Estimated rows ≠ actual (10x+) | Stale statistics | Run `ANALYZE table_name;` |
| `Sort` with high cost | No index matching `ORDER BY` | Add index covering sort columns |
| `Nested Loop` with large inner | Wrong join strategy | Verify indexes on join columns; increase `work_mem` |
| `Hash Join` spilling to disk | `work_mem` too small | Increase `work_mem` |
| High `Buffers: read` count | Data not in cache | Increase `shared_buffers`; check for bloated tables |

### Iterative Query Optimisation Process

```
1. [ ] Identify the slow query (app logs, pg_stat_statements, slow query log)
2. [ ] Run EXPLAIN ANALYZE
3. [ ] Identify the most expensive node
4. [ ] Determine root cause (missing index, bad stats, inefficient join, sort)
5. [ ] Apply ONE fix at a time
6. [ ] Re-run EXPLAIN ANALYZE and compare
7. [ ] Repeat until acceptable performance
8. [ ] Verify the fix doesn't degrade other queries
```

### MySQL-Specific EXPLAIN

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;

-- Key columns:
-- type: ALL (full scan) → index → range → ref → eq_ref → const → system
-- possible_keys: indexes the optimiser considered
-- key: index actually used
-- rows: estimated rows scanned
-- Extra: "Using filesort", "Using temporary" are red flags

-- MySQL 8.0+:
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;
```

---

## 9. ORM Integration Guidelines

### ORM Selection by Stack

| Stack | Primary ORM | Alternative |
|---|---|---|
| **PHP (Symfony)** | Doctrine ORM | — |
| **PHP (Laravel)** | Eloquent ORM | — |
| **Node.js (NestJS)** | TypeORM or MikroORM | Prisma |
| **Node.js (General)** | Prisma | TypeORM, Drizzle |
| **.NET** | Entity Framework Core | Dapper (micro-ORM) |
| **Java (Spring)** | Hibernate / Spring Data JPA | jOOQ (SQL-first) |
| **Go** | GORM | sqlx, Ent |

### ORM Best Practices

**1. Always review generated SQL.**

```typescript
// TypeORM
const dataSource = new DataSource({ logging: ['query', 'error'] });

// Prisma
const prisma = new PrismaClient({ log: ['query', 'warn', 'error'] });
```

**2. Solve N+1 queries with eager loading.**

```typescript
// TypeORM
const orders = await orderRepository.find({
    relations: ['customer', 'items', 'items.product'],
    where: { status: 'active' },
});

// Prisma
const orders = await prisma.order.findMany({
    where: { status: 'active' },
    include: {
        customer: true,
        items: { include: { product: true } },
    },
});
```

**3. Use raw SQL for complex queries.**

```typescript
// TypeORM — SAFE raw query
const results = await dataSource.query(`
    SELECT u.id, u.email, COUNT(o.id) AS order_count
    FROM users u
    LEFT JOIN orders o ON o.customer_id = u.id AND o.status = 'completed'
    WHERE u.deleted_at IS NULL
    GROUP BY u.id, u.email
    HAVING COUNT(o.id) > 5
    ORDER BY order_count DESC
    LIMIT $1
`, [50]);

// Prisma — raw query
const results = await prisma.$queryRaw`
    SELECT u.id, u.email, COUNT(o.id) AS order_count
    FROM users u
    LEFT JOIN orders o ON o.customer_id = u.id AND o.status = 'completed'
    WHERE u.deleted_at IS NULL
    GROUP BY u.id, u.email
    HAVING COUNT(o.id) > 5
    ORDER BY order_count DESC
    LIMIT ${50}
`;
```

**4. ORM Migration Workflow.**
- Generate migrations from entity/model changes, never edit DB directly.
- Review generated migration SQL before applying.
- Test migrations against a copy of production data.

**5. Repository Pattern.**

```
Service → Repository Interface → Repository Implementation (uses ORM)
```

Keeps business logic decoupled from data access. Makes testing easier (mock the repository interface).

---

## 10. Database Maintenance & Monitoring

### Statistics and Vacuuming (PostgreSQL)

```sql
-- Update table statistics
ANALYZE orders;

-- Check for table bloat
SELECT relname, n_dead_tup, n_live_tup,
       ROUND(n_dead_tup::NUMERIC / NULLIF(n_live_tup, 0) * 100, 2) AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;
```

### Slow Query Identification

```sql
-- PostgreSQL: pg_stat_statements
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

SELECT
    calls,
    ROUND(total_exec_time::NUMERIC, 2) AS total_ms,
    ROUND(mean_exec_time::NUMERIC, 2) AS mean_ms,
    ROUND(max_exec_time::NUMERIC, 2) AS max_ms,
    LEFT(query, 200) AS query_preview
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;
```

```sql
-- MySQL: slow query log
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;
```

### Connection Pooling

- Always use connection pooling in production.
- PostgreSQL: PgBouncer or built-in pooling.
- MySQL: ProxySQL or driver-level pooling.
- Pool size formula: `pool_size = (core_count * 2) + effective_spindle_count`.
- Alert when pool utilisation exceeds 80%.
