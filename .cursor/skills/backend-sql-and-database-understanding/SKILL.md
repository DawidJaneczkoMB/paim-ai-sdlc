---
name: sql-and-database-understanding
description: Use when designing database schemas, writing performant SQL queries, normalisation strategies, indexing, joins optimisation, locking mechanics, transactions, query debugging with EXPLAIN, and ORM integration. Applies to PostgreSQL, MySQL, MariaDB, SQL Server, and Oracle. Covers ORM usage with TypeORM, Prisma, Doctrine, Eloquent, Entity Framework, Hibernate, and GORM.
---

# SQL & Database Engineering

Standards, patterns, and procedures for database schema design, performant SQL, query debugging, and database engineering. Database-engine-agnostic with engine-specific notes where critical.

For full detailed reference on each topic, see [reference.md](reference.md).

## Guiding Principles

| Principle                     | Application                                                                                                                      |
| ----------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| **Data Integrity First**      | Enforce constraints at the database level (NOT NULL, UNIQUE, FK, CHECK). Never rely solely on application-level validation.      |
| **Least Privilege**           | Database users/roles should have only the permissions they need. Application connections should never use the superuser account. |
| **Explicit Over Implicit**    | Always specify column lists in SELECT, INSERT, and JOIN clauses. Avoid `SELECT *` in production code.                            |
| **Measure Before Optimising** | Use `EXPLAIN ANALYZE` to identify actual bottlenecks before adding indexes or restructuring queries.                             |
| **Schema as Code**            | All schema changes go through versioned migration files. Never modify production schemas manually.                               |

## Quick Reference

### Naming Conventions

| Element            | Convention                                | Example                         |
| ------------------ | ----------------------------------------- | ------------------------------- |
| Tables             | `snake_case`, plural nouns                | `order_items`, `user_addresses` |
| Columns            | `snake_case`                              | `first_name`, `created_at`      |
| Primary keys       | `id` (preferred) or `<table_singular>_id` | `id`, `user_id`                 |
| Foreign keys       | `<referenced_table_singular>_id`          | `user_id`, `order_id`           |
| Indexes            | `idx_<table>_<columns>`                   | `idx_orders_user_id_status`     |
| Unique constraints | `uq_<table>_<columns>`                    | `uq_users_email`                |
| Check constraints  | `chk_<table>_<description>`               | `chk_orders_total_positive`     |
| Junction tables    | `<table1>_<table2>` (alphabetical)        | `products_tags`, `roles_users`  |

### Standard Columns

Every table **MUST** include:

```sql
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- ... domain columns ...
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
```

### Data Types — Choose Precisely

| Need                     | Use                                            | Avoid                                  |
| ------------------------ | ---------------------------------------------- | -------------------------------------- |
| Monetary values          | `NUMERIC(19,4)` or `DECIMAL(19,4)`             | `FLOAT`, `DOUBLE` (precision loss)     |
| Timestamps               | `TIMESTAMP WITH TIME ZONE`                     | `TIMESTAMP` without timezone           |
| Boolean flags            | `BOOLEAN`                                      | `TINYINT`, `CHAR(1)`                   |
| Short text (name, email) | `VARCHAR(n)` with appropriate limit            | Unbounded `TEXT` for structured fields |
| Long text (descriptions) | `TEXT`                                         | `VARCHAR(10000)`                       |
| Enums                    | `VARCHAR` with CHECK constraint or native ENUM | Magic integers                         |
| JSON/semi-structured     | `JSONB` (PostgreSQL) or `JSON`                 | Storing relational data as JSON        |
| IP addresses             | `INET` (PostgreSQL) or `VARCHAR(45)`           | `VARCHAR(15)` (IPv6 won't fit)         |

### Composite Index Column Order

1. **Equality conditions first** (`WHERE status = 'active'`)
2. **Range conditions second** (`WHERE created_at > '2025-01-01'`)
3. **Sort columns last** (`ORDER BY created_at DESC`)

### Query Writing Rules

| Rule                               | Do                                         | Don't                                                 |
| ---------------------------------- | ------------------------------------------ | ----------------------------------------------------- |
| Specify columns                    | `SELECT id, name, email FROM users`        | `SELECT * FROM users`                                 |
| Use parameterised queries          | `WHERE id = $1` / `WHERE id = ?`           | `WHERE id = '` + userId + `'` (SQL injection)         |
| Limit results                      | `LIMIT 100` or paginate                    | Unbounded `SELECT` on large tables                    |
| Filter early                       | `WHERE` clause narrows rows before joins   | Joining full tables then filtering                    |
| Use EXISTS over IN for subqueries  | `WHERE EXISTS (SELECT 1 FROM ...)`         | `WHERE id IN (SELECT id FROM ...)` on large sets      |
| Avoid functions on indexed columns | `WHERE created_at >= '2025-01-01'`         | `WHERE DATE(created_at) = '2025-01-01'` (kills index) |
| Use UNION ALL over UNION           | `UNION ALL` when duplicates are acceptable | `UNION` (forces sort + dedup)                         |

### Common Anti-Patterns

| Anti-Pattern                           | Problem                                                 | Fix                                               |
| -------------------------------------- | ------------------------------------------------------- | ------------------------------------------------- |
| `SELECT *`                             | Fetches unnecessary columns, breaks when schema changes | Explicitly list needed columns                    |
| `N+1 queries`                          | 1 query to fetch parents + N queries for children       | Use `JOIN` or batch loading (`WHERE id IN (...)`) |
| `LIKE '%value%'`                       | Leading wildcard prevents index use                     | Use full-text search (`tsvector` / `FULLTEXT`)    |
| `ORDER BY RAND()`                      | Scans and sorts entire table                            | Use sampled approach                              |
| Implicit type conversion               | `WHERE varchar_col = 123` may skip index                | Match types explicitly                            |
| Correlated subquery in SELECT          | Executes subquery for every row                         | Rewrite as `JOIN` or window function              |
| Missing `LIMIT` on exploratory queries | Accidentally fetching millions of rows                  | Always add `LIMIT`                                |

### EXPLAIN Output — Key Fields

| Field                        | What It Tells You                                                  |
| ---------------------------- | ------------------------------------------------------------------ |
| **Seq Scan**                 | Full table scan — often a sign of missing index                    |
| **Index Scan**               | Using an index — this is what you want                             |
| **Index Only Scan**          | Data served entirely from the index (covering index) — best case   |
| **Nested Loop**              | For each row in outer, scan inner — efficient for small outer sets |
| **Hash Join**                | Build hash table, probe with other — efficient for large equijoins |
| **Sort**                     | Explicit sort operation — check if an index could eliminate this   |
| **Actual Rows vs Estimated** | Compare to find planner misestimates                               |

### Optimistic vs Pessimistic Locking

| Factor               | Optimistic                  | Pessimistic                        |
| -------------------- | --------------------------- | ---------------------------------- |
| Conflict frequency   | Low (< 1% of operations)    | High or unpredictable              |
| Read/write ratio     | Read-heavy                  | Write-heavy on contested resources |
| Transaction duration | Can be long (no locks held) | Must be short (locks block others) |
| Failure handling     | Retry on conflict           | Block until lock is released       |
| Distributed systems  | Preferred                   | Difficult                          |

## Implementation Procedure

When writing SQL or designing a database schema, follow this workflow:

```
Implementation progress:
- [ ] Step 1: Understand the data requirements
- [ ] Step 2: Design the schema (entities, relationships, constraints)
- [ ] Step 3: Choose normalisation level and document any denormalisation decisions
- [ ] Step 4: Define indexes based on expected query patterns
- [ ] Step 5: Write the migration(s)
- [ ] Step 6: Write the queries (raw SQL or ORM)
- [ ] Step 7: Run EXPLAIN ANALYZE on critical queries
- [ ] Step 8: Implement appropriate locking and transaction strategy
- [ ] Step 9: Review ORM-generated SQL (if applicable)
- [ ] Step 10: Test with realistic data volumes
```

**Step 1: Understand the data requirements**
Read the task description and acceptance criteria. Identify the entities, their attributes, and the relationships between them. Clarify cardinality (one-to-many, many-to-many).

**Step 2: Design the schema**
Create tables following the naming conventions and standard columns defined in this skill. Define relationships with appropriate foreign keys and ON DELETE actions. See [reference.md](reference.md) § Schema Architecture Design.

**Step 3: Choose normalisation level**
Target 3NF by default. Document any intentional denormalisation with a clear justification tied to measured performance needs or snapshot requirements. See [reference.md](reference.md) § Normalisation Strategies.

**Step 4: Define indexes**
Identify the primary query patterns and create indexes to support them. Index all foreign keys. Create composite indexes for multi-column filters and sorts. See [reference.md](reference.md) § Indexes.

**Step 5: Write the migration(s)**
Generate migration files with both `up` and `down` methods. Review the generated DDL before applying.

**Step 6: Write the queries**
Write queries following the performance and formatting rules. Use parameterised queries. Avoid `SELECT *`. See [reference.md](reference.md) § Writing Performant SQL Queries.

**Step 7: Run EXPLAIN ANALYZE**
Verify critical query plans. Look for sequential scans on large tables, missing indexes, and planner misestimates.

**Step 8: Implement locking and transaction strategy**
Choose optimistic or pessimistic locking based on the contention profile. Define transaction boundaries and appropriate isolation levels. See [reference.md](reference.md) § Transactions and Locking Mechanics.

**Step 9: Review ORM-generated SQL**
Enable query logging and inspect generated queries for N+1 problems, unnecessary joins, or missing eager loading. See [reference.md](reference.md) § ORM Integration.

**Step 10: Test with realistic data volumes**
Seed the development database with production-scale data. Verify that queries perform within acceptable time limits under load.

## Connected Skills

- `architecture-design` — for designing data models as part of broader system architecture
- `code-review` — for validating SQL quality, index coverage, and query performance
- `technical-context-discovery` — for establishing database conventions and existing patterns before designing
