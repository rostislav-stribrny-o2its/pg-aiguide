---
name: design-postgres-tables
description: |
  Use this skill for general PostgreSQL table design.

  **Trigger when user asks to:**
  - Design PostgreSQL tables, schemas, or data models when creating new tables and when modifying existing ones.
  - Choose data types, constraints, or indexes for PostgreSQL
  - Create user tables, order tables, reference tables, or JSONB schemas
  - Understand PostgreSQL best practices for normalization, constraints, or indexing
  - Design update-heavy, upsert-heavy, or OLTP-style tables


  **Keywords:** PostgreSQL schema, table design, data types, PRIMARY KEY, FOREIGN KEY, indexes, B-tree, GIN, JSONB, constraints, normalization, BCNF, lookup tables, identity columns, partitioning, row-level security

  Comprehensive reference covering data types, indexing strategies, constraints, JSONB patterns, partitioning, and PostgreSQL-specific best practices.
license: Apache-2.0
metadata:
  author: tigerdata
---

# PostgreSQL Table Design

Target: **PostgreSQL 18**. The schema is changed only by Flyway migrations (see the `flyway-migrations` skill); the application accesses it with Spring Data JDBC and `JdbcClient`.

## Core Rules

- **Naming:** `snake_case`, singular table names (`customer`, `customer_order`; avoid reserved words such as `user`, `order`); primary key column `id`; foreign key column `<referenced_table>_id` (e.g. `order_status_id` → `order_status (id)`).
- Define a **PRIMARY KEY** for every entity table (users, orders, etc.). Not always needed for time-series/event/log data. Use `BIGINT GENERATED ALWAYS AS IDENTITY (MAXVALUE 9007199254740991)`: the upper bound 2^53−1 keeps every ID exact as a JSON number in a JavaScript frontend. Do not use `UUID` as a primary key; if an opaque public identifier is really needed, add it as a separate `UNIQUE` column.
- **Normalize to at least BCNF, aim for 5NF** in the logical model, to eliminate redundancy and update anomalies; denormalize **only** for measured, high-ROI reads where join performance is proven problematic, and record why. Premature denormalization creates maintenance burden.
- **Enumerated values are lookup tables** with a foreign key (`order_status_id smallint REFERENCES order_status (id)`), their rows shipped in Flyway migrations — not `CREATE TYPE … AS ENUM`, and not a free-text column with `CHECK (… IN (…))`.
- **No business logic in the database.** Declarative constraints (PK, FK, UNIQUE, CHECK, EXCLUDE, NOT NULL) belong here; business rules, workflows and computed decisions belong in the application service layer. No triggers or stored procedures for business logic.
- Add **NOT NULL** everywhere it’s semantically required; use **DEFAULT**s for common values.
- Create **indexes for access paths you actually query**: PK/unique (auto), **FK columns (manual!)**, frequent filters/sorts, and join keys.
- Prefer **TIMESTAMPTZ** for event time; **NUMERIC(p,s)** for money and every exact business quantity (mapped to `BigDecimal`); **TEXT** for strings; **BIGINT** for IDs and large counters, **INTEGER** where the range is known to fit.

## PostgreSQL “Gotchas”

- **Identifiers**: unquoted → lowercased. Avoid quoted/mixed-case names. Convention: use `snake_case` for table/column names.
- **Unique + NULLs**: UNIQUE allows multiple NULLs. Use `UNIQUE (...) NULLS NOT DISTINCT` (PG15+) to restrict to one NULL.
- **FK indexes**: PostgreSQL **does not** auto-index FK columns. Add them.
- **No silent coercions**: length/precision overflows error out (no truncation). Example: inserting 999 into `NUMERIC(2,0)` fails with error, unlike some databases that silently truncate or round.
- **Sequences/identity have gaps** (normal; don't "fix"). Rollbacks, crashes, and concurrent transactions create gaps in ID sequences (1, 2, 5, 6...). This is expected behavior—don't try to make IDs consecutive.
- **Heap storage**: no clustered PK by default (unlike SQL Server/MySQL InnoDB); `CLUSTER` is one-off reorganization, not maintained on subsequent inserts. Row order on disk is insertion order unless explicitly clustered.
- **MVCC**: updates/deletes leave dead tuples; vacuum handles them—design to avoid hot wide-row churn.

## Data Types

- **IDs**: `BIGINT GENERATED ALWAYS AS IDENTITY (MAXVALUE 9007199254740991)` (mapped to `Long`, OpenAPI `integer/int64` with `maximum: 9007199254740991`). Lookup tables use a small explicit key (`smallint`) assigned in the migration. A non-key opaque identifier, when needed, is a `UUID` generated with `uuidv7()` (PG18).
- **Integers**: `INTEGER` for counts and quantities with a known range; `BIGINT` for IDs and values that can exceed 2^31 — any `BIGINT` value sent to a JavaScript frontend must stay ≤ 2^53−1 (add a CHECK if it is not an identity); `SMALLINT` for lookup-table keys.
- **Floats**: not for money, quantities, rates or anything compared for equality — use `NUMERIC(p,s)`. `DOUBLE PRECISION` only for genuinely approximate values (scientific measurements, statistics) when the project decides so explicitly; never `REAL`.
- **Strings**: prefer `TEXT`; if length limits needed, use `CHECK (LENGTH(col) <= n)` instead of `VARCHAR(n)`; avoid `CHAR(n)`. Use `BYTEA` for binary data. Large strings/binary (>2KB default threshold) automatically stored in TOAST with compression. TOAST storage: `PLAIN` (no TOAST), `EXTENDED` (compress + out-of-line), `EXTERNAL` (out-of-line, no compress), `MAIN` (compress, keep in-line if possible). Default `EXTENDED` usually optimal. Control with `ALTER TABLE tbl ALTER COLUMN col SET STORAGE strategy` and `ALTER TABLE tbl SET (toast_tuple_target = 4096)` for threshold. Case-insensitive: for locale/accent handling use non-deterministic collations; for plain ASCII use expression indexes on `LOWER(col)` (preferred unless column needs case-insensitive PK/FK/UNIQUE) or `CITEXT`.
- **Money**: `NUMERIC(p,s)` (never float).
- **Time**: `TIMESTAMPTZ` for timestamps; `DATE` for date-only; `INTERVAL` for durations. Avoid `TIMESTAMP` (without timezone). Use `now()` for transaction start time, `clock_timestamp()` for current wall-clock time.
- **Booleans**: `BOOLEAN` with `NOT NULL` constraint unless tri-state values are required.
- **Enums**: lookup table with FK (see Core Rules), even for small, stable sets. Lookup values are data: they can carry names, ordering and validity, and they change by a migration, not by altering a type.
- **Arrays**: `TEXT[]`, `INTEGER[]`, etc. Use for ordered lists where you query elements. Index with **GIN** for containment (`@>`, `<@`) and overlap (`&&`) queries. Access: `arr[1]` (1-indexed), `arr[1:3]` (slicing). Good for tags, categories; avoid for relations—use junction tables instead. Literal syntax: `'{val1,val2}'` or `ARRAY[val1,val2]`.
- **Range types**: `daterange`, `numrange`, `tstzrange` for intervals. Support overlap (`&&`), containment (`@>`), operators. Index with **GiST**. Good for scheduling, versioning, numeric ranges. Pick a bounds scheme and use it consistently; prefer `[)` (inclusive/exclusive) by default.
- **Network types**: `INET` for IP addresses, `CIDR` for network ranges, `MACADDR` for MAC addresses. Support network operators (`<<`, `>>`, `&&`).
- **Geometric types**: avoid `POINT`, `LINE`, `POLYGON`, `CIRCLE`. Index with **GiST**. Consider **PostGIS** for spatial features.
- **Text search**: `TSVECTOR` for full-text search documents, `TSQUERY` for search queries. Index `tsvector` with **GIN**. Always specify language: `to_tsvector('english', col)` and `to_tsquery('english', 'query')`. Never use single-argument versions. This applies to both index expressions and queries.
- **Domain types**: `CREATE DOMAIN email AS TEXT CHECK (VALUE ~ '^[^@]+@[^@]+$')` for reusable custom types with validation. Enforces constraints across tables.
- **Composite types**: `CREATE TYPE address AS (street TEXT, city TEXT, zip TEXT)` for structured data within columns. Access with `(col).field` syntax.
- **JSONB**: preferred over JSON; index with **GIN**. Use only for optional/semi-structured attrs. ONLY use JSON if the original ordering of the contents MUST be preserved.
- **Vector types**: `vector` type by `pgvector` for vector similarity search for embeddings.

### Do not use the following data types

- DO NOT use `timestamp` (without time zone); DO use `timestamptz` instead.
- DO NOT use `char(n)` or `varchar(n)`; DO use `text` instead.
- DO NOT use `money` type; DO use `numeric` instead.
- DO NOT use `timetz` type; DO use `timestamptz` instead.
- DO NOT use `timestamptz(0)` or any other precision specification; DO use `timestamptz` instead
- DO NOT use `serial` type; DO use `generated always as identity` instead.
- DO NOT use `POINT`, `LINE`, `POLYGON`, `CIRCLE` built-in types, DO use `geometry` from postgis extension instead.

## Table Types

- **Regular**: default; fully durable, logged.
- **TEMPORARY**: session-scoped, auto-dropped, not logged. Faster for scratch work.
- **UNLOGGED**: persistent but not crash-safe. Faster writes; good for caches/staging.

## Row-Level Security

Enable with `ALTER TABLE tbl ENABLE ROW LEVEL SECURITY`. Create policies: `CREATE POLICY user_access ON orders FOR SELECT TO app_users USING (user_id = current_user_id())`. Built-in user-based access control at the row level.

Use RLS only when the project decides on it explicitly (for example several database consumers that must see different rows). By default authorization lives in the application; enabling RLS as well means two places of authorization and tests at the database level.

## Constraints

- **PK**: implicit UNIQUE + NOT NULL; creates a B-tree index.
- **FK**: specify `ON DELETE/UPDATE` action (`CASCADE`, `RESTRICT`, `SET NULL`, `SET DEFAULT`). Add explicit index on referencing column—speeds up joins and prevents locking issues on parent deletes/updates. Use `DEFERRABLE INITIALLY DEFERRED` for circular FK dependencies checked at transaction end.
- **UNIQUE**: creates a B-tree index; allows multiple NULLs unless `NULLS NOT DISTINCT` (PG15+). Standard behavior: `(1, NULL)` and `(1, NULL)` are allowed. With `NULLS NOT DISTINCT`: only one `(1, NULL)` allowed. Prefer `NULLS NOT DISTINCT` unless you specifically need duplicate NULLs.
- **CHECK**: row-local constraints; NULL values pass the check (three-valued logic). Example: `CHECK (price > 0)` allows NULL prices. Combine with `NOT NULL` to enforce: `price NUMERIC NOT NULL CHECK (price > 0)`.
- **EXCLUDE**: prevents overlapping values using operators. `EXCLUDE USING gist (room_id WITH =, booking_period WITH &&)` prevents double-booking rooms. Requires appropriate index type (often GiST).

## Indexing

- **B-tree**: default for equality/range queries (`=`, `<`, `>`, `BETWEEN`, `ORDER BY`)
- **Composite**: order matters—index used if equality on leftmost prefix (`WHERE a = ? AND b > ?` uses index on `(a,b)`, but `WHERE b = ?` does not). Put most selective/frequently filtered columns first.
- **Covering**: `CREATE INDEX ON tbl (id) INCLUDE (name, email)` - includes non-key columns for index-only scans without visiting table.
- **Partial**: for hot subsets (`WHERE status = 'active'` → `CREATE INDEX ON tbl (user_id) WHERE status = 'active'`). Any query with `status = 'active'` can use this index.
- **Expression**: for computed search keys (`CREATE INDEX ON tbl (LOWER(email))`). Expression must match exactly in WHERE clause: `WHERE LOWER(email) = 'user@example.com'`.
- **GIN**: JSONB containment/existence, arrays (`@>`, `?`), full-text search (`@@`)
- **GiST**: ranges, geometry, exclusion constraints
- **BRIN**: very large, naturally ordered data (time-series)—minimal storage overhead. Effective when row order on disk correlates with indexed column (insertion order or after `CLUSTER`).

## Partitioning

- Use for very large tables (>100M rows) where queries consistently filter on partition key (often time/date).
- Alternate use: use for tables where data maintenance tasks dictates e.g. data pruned or bulk replaced periodically
- **RANGE**: common for time-series (`PARTITION BY RANGE (created_at)`). Create partitions: `CREATE TABLE logs_2024_01 PARTITION OF logs FOR VALUES FROM ('2024-01-01') TO ('2024-02-01')`.
- **LIST**: for discrete values (`PARTITION BY LIST (region)`). Example: `FOR VALUES IN ('us-east', 'us-west')`.
- **HASH**: for even distribution when no natural key (`PARTITION BY HASH (user_id)`). Creates N partitions with modulus.
- **Constraint exclusion**: requires `CHECK` constraints on partitions for query planner to prune. Auto-created for declarative partitioning (PG10+).
- Prefer declarative partitioning. Do NOT use table inheritance.
- **Limitations**: no global UNIQUE constraints—include partition key in PK/UNIQUE. Foreign keys from and to partitioned tables are supported in current PostgreSQL versions (to a partitioned table: the referenced key must include the partition key); do not emulate them with triggers.

## Special Considerations

### Update-Heavy Tables

- **Separate hot/cold columns**—put frequently updated columns in separate table to minimize bloat.
- **Use `fillfactor=90`** to leave space for HOT updates that avoid index maintenance.
- **Avoid updating indexed columns**—prevents beneficial HOT updates.
- **Partition by update patterns**—separate frequently updated rows in a different partition from stable data.

### Insert-Heavy Workloads

- **Minimize indexes**—only create what you query; every index slows inserts.
- **Use `COPY` or multi-row `INSERT`** instead of single-row inserts.
- **UNLOGGED tables** for rebuildable staging data—much faster writes.
- **Defer index creation** for bulk loads—>drop index, load data, recreate indexes.
- **Partition by time/hash** to distribute load.
- **Use a natural key for primary key** such as a (timestamp, device_id) if enforcing global uniqueness is important many insert-heavy tables don't need a primary key at all.
- If you do need a surrogate key, **Prefer `BIGINT GENERATED ALWAYS AS IDENTITY` over `UUID`**.

### Upsert-Friendly Design

- **Requires UNIQUE index** on conflict target columns—`ON CONFLICT (col1, col2)` needs exact matching unique index (partial indexes don't work).
- **Use `EXCLUDED.column`** to reference would-be-inserted values; only update columns that actually changed to reduce write overhead.
- **`DO NOTHING` faster** than `DO UPDATE` when no actual update needed.

### Safe Schema Evolution

- **Transactional DDL**: most DDL operations can run in transactions and be rolled back—`BEGIN; ALTER TABLE...; ROLLBACK;` for safe testing. Every schema change still ships as a Flyway migration; nothing is applied by hand.
- **Concurrent index creation**: `CREATE INDEX CONCURRENTLY` avoids blocking writes but can't run in transactions.
- **Volatile defaults cause rewrites**: adding columns with volatile defaults (e.g., `gen_random_uuid()`, `clock_timestamp()`, `random()`) rewrites the entire table. Non-volatile defaults — constants and stable functions such as `now()` — are fast (existing rows get the value computed once).
- **Drop constraints before columns**: `ALTER TABLE DROP CONSTRAINT` then `DROP COLUMN` to avoid dependency issues.
- **Function signature changes**: `CREATE OR REPLACE` with different arguments creates overloads, not replacements. DROP old version if no overload desired.

## Generated Columns

- `... GENERATED ALWAYS AS (<expr>) STORED` for computed, indexable fields. PG18+ adds `VIRTUAL` columns (computed on read, not stored).

## Extensions

- **`pgcrypto`**: digests and encryption functions. Do not store or hash user passwords in the application database — authentication belongs to the identity provider.
- **`uuid-ossp`**: not needed on PG18 (`uuidv7()`, `gen_random_uuid()` are built in).
- **`pg_trgm`**: fuzzy text search with `%` operator, `similarity()` function. Index with GIN for `LIKE '%pattern%'` acceleration.
- **`citext`**: case-insensitive text type. Prefer expression indexes on `LOWER(col)` unless you need case-insensitive constraints.
- **`btree_gin`/`btree_gist`**: enable mixed-type indexes (e.g., GIN index on both JSONB and text columns).
- **`hstore`**: key-value pairs; mostly superseded by JSONB but useful for simple string mappings.
- **`postgis`**: comprehensive geospatial support beyond basic geometric types—essential for location-based applications.
- **`pgvector`**: vector similarity search for embeddings.
- **`pgaudit`**: audit logging for all database activity.

## JSONB Guidance

- Prefer `JSONB` with **GIN** index.
- Default: `CREATE INDEX ON tbl USING GIN (jsonb_col);` → accelerates:
  - **Containment** `jsonb_col @> '{"k":"v"}'`
  - **Key existence** `jsonb_col ? 'k'`, **any/all keys** `?\|`, `?&`
  - **Path containment** on nested docs
  - **Disjunction** `jsonb_col @> ANY(ARRAY['{"status":"active"}', '{"status":"pending"}'])`
- Heavy `@>` workloads: consider opclass `jsonb_path_ops` for smaller/faster containment-only indexes:
  - `CREATE INDEX ON tbl USING GIN (jsonb_col jsonb_path_ops);`
  - **Trade-off**: loses support for key existence (`?`, `?|`, `?&`) queries—only supports containment (`@>`)
- Equality/range on a specific scalar field: extract and index with B-tree (generated column or expression):
  - `ALTER TABLE tbl ADD COLUMN price INT GENERATED ALWAYS AS ((jsonb_col->>'price')::INT) STORED;`
  - `CREATE INDEX ON tbl (price);`
  - Prefer queries like `WHERE price BETWEEN 100 AND 500` (uses B-tree) over `WHERE (jsonb_col->>'price')::INT BETWEEN 100 AND 500` without index.
- Arrays inside JSONB: use GIN + `@>` for containment (e.g., tags). Consider `jsonb_path_ops` if only doing containment.
- Keep core relations in tables; use JSONB for optional/variable attributes.
- Use constraints to limit allowed JSONB values in a column e.g. `config JSONB NOT NULL CHECK(jsonb_typeof(config) = 'object')`

## Examples

### Customer

```sql
CREATE TABLE customer (
  id BIGINT GENERATED ALWAYS AS IDENTITY (MAXVALUE 9007199254740991) PRIMARY KEY,
  email TEXT NOT NULL UNIQUE CHECK (length(email) <= 254),
  name TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ON customer (LOWER(email));
CREATE INDEX ON customer (created_at);
```

### Order with lookup table

```sql
CREATE TABLE order_status (
  id SMALLINT PRIMARY KEY,                            -- explicit, assigned in the migration
  code TEXT NOT NULL UNIQUE CHECK (length(code) <= 30),
  name TEXT NOT NULL
);
INSERT INTO order_status (id, code, name) VALUES (1, 'PENDING', 'Pending'), (2, 'PAID', 'Paid'), (3, 'CANCELED', 'Canceled');

CREATE TABLE customer_order (
  id BIGINT GENERATED ALWAYS AS IDENTITY (MAXVALUE 9007199254740991) PRIMARY KEY,
  customer_id BIGINT NOT NULL REFERENCES customer(id),
  order_status_id SMALLINT NOT NULL DEFAULT 1 REFERENCES order_status(id),
  total NUMERIC(19,2) NOT NULL CHECK (total > 0),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  version BIGINT NOT NULL DEFAULT 0                    -- optimistic locking (@Version)
);
CREATE INDEX ON customer_order (customer_id);
CREATE INDEX ON customer_order (order_status_id);
CREATE INDEX ON customer_order (created_at);
```

### JSONB

```sql
CREATE TABLE customer_profile (
  customer_id BIGINT PRIMARY KEY REFERENCES customer(id),
  attrs JSONB NOT NULL DEFAULT '{}',
  theme TEXT GENERATED ALWAYS AS (attrs->>'theme') STORED
);
CREATE INDEX customer_profile_attrs_gin ON customer_profile USING GIN (attrs);
```

## Provenance

Modified for the Elysium Java base (PostgreSQL 18, Flyway, Spring Data JDBC) — this file has been changed from the original.
Source: [timescale/pg-aiguide](https://github.com/timescale/pg-aiguide), `skills/design-postgres-tables`,
Apache License 2.0 (see the repository `LICENSE` and `NOTICE`: Copyright 2025 Timescale, Inc., d/b/a Tiger Data).
Changes: bounded `bigint` identities (2^53−1) instead of UUID keys, BCNF/5NF, lookup tables instead of ENUM/CHECK lists,
no business logic or password hashing in the database, RLS only by explicit decision, outdated partition FK note corrected,
TimescaleDB references removed, `NUMERIC` for exact quantities.
