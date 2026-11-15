# PostgreSQL reference (non-binding)

Part of `db-design-postgres-tables`. General PostgreSQL 18 knowledge for choosing among options.
**Not binding:** where anything here differs from the skill's "Binding rules", the binding rules
win. Partitioning, extensions, special index types and table types are used only when a measured
need or an explicit project decision asks for them.

## Data types in detail

- **Integers**: `INTEGER` for counts and quantities with a known range; `BIGINT` for values that can
  exceed 2^31 — any `BIGINT` value sent to a JavaScript frontend must stay ≤ 2^53−1 (add a CHECK if
  it is not an identity); `SMALLINT` for lookup-table keys.
- **Floats**: `DOUBLE PRECISION` only for genuinely approximate values (scientific measurements,
  statistics) when the project decides so explicitly; never `REAL`.
- **Strings**: `BYTEA` for binary data. Large values (>2KB) are stored in TOAST with compression;
  strategies `PLAIN`, `EXTENDED` (default), `EXTERNAL`, `MAIN`
  (`ALTER TABLE tbl ALTER COLUMN col SET STORAGE …`, `toast_tuple_target`). Case-insensitive: for
  locale/accent handling non-deterministic collations; for plain ASCII expression indexes on
  `LOWER(col)` (preferred unless the column needs case-insensitive PK/FK/UNIQUE) or `CITEXT`.
- **Time**: `INTERVAL` for durations. `now()` = transaction start, `clock_timestamp()` = wall clock.
- **Arrays**: `TEXT[]`, `INTEGER[]` for ordered lists whose elements you query; GIN for `@>`, `<@`,
  `&&`. Never for relations — junction tables.
- **Range types**: `daterange`, `numrange`, `tstzrange`; overlap `&&`, containment `@>`; GiST index;
  prefer `[)` bounds.
- **Network types**: `INET`, `CIDR`, `MACADDR`.
- **Geometric types**: not the built-in `POINT`/`LINE`/`POLYGON`/`CIRCLE` — PostGIS `geometry`.
- **Text search**: `TSVECTOR`/`TSQUERY` with GIN; always the two-argument form with a language.
- **Domain types**: `CREATE DOMAIN email AS TEXT CHECK (VALUE ~ '^[^@]+@[^@]+$')`.
- **Composite types**: `CREATE TYPE address AS (street TEXT, city TEXT, zip TEXT)`.
- **JSONB** (not JSON unless key order must be kept) only for optional/semi-structured attributes.
- **Vector**: `vector` from `pgvector` for embeddings.
- A non-key opaque public identifier, only when really needed: `UUID` from `uuidv7()` (PG18) in a
  separate `UNIQUE` column — never the primary key.

## Table types

- **Regular**: default; durable, logged.
- **TEMPORARY**: session-scoped, auto-dropped, not logged.
- **UNLOGGED**: persistent but not crash-safe; for rebuildable caches/staging.

## Indexing

- **B-tree**: default for `=`, `<`, `>`, `BETWEEN`, `ORDER BY`.
- **Composite**: used when the leftmost prefix is constrained (`(a,b)` serves `a = ? AND b > ?`, not
  `b = ?`); most selective / most filtered first.
- **Covering**: `CREATE INDEX … ON tbl (id) INCLUDE (name, email)` for index-only scans.
- **Partial**: `CREATE INDEX … ON tbl (user_id) WHERE status = 'active'` for hot subsets.
- **Expression**: `CREATE INDEX … ON tbl (LOWER(email))`; the query must use the same expression.
- **GIN**: JSONB, arrays, full text. **GiST**: ranges, geometry, exclusion constraints.
- **BRIN**: very large, naturally ordered data.

## Partitioning

- For very large tables (>100M rows) whose queries filter on the partition key, or where data is
  pruned or bulk-replaced periodically.
- RANGE (time), LIST (discrete values), HASH (even spread). Declarative only, never inheritance.
- No global UNIQUE: include the partition key in PK/UNIQUE. Foreign keys to and from partitioned
  tables are supported (to one: the referenced key includes the partition key); never emulate them
  with triggers.

## Special workloads

- **Update-heavy**: separate hot and cold columns; `fillfactor=90` for HOT updates; avoid updating
  indexed columns.
- **Insert-heavy**: minimal indexes; `COPY` or multi-row `INSERT`; UNLOGGED for rebuildable staging;
  drop and recreate indexes around bulk loads; surrogate keys as `BIGINT` identity, not `UUID`.
- **Upsert**: `ON CONFLICT (cols)` needs a matching unique index (partial indexes do not work); use
  `EXCLUDED.col`; `DO NOTHING` is cheaper than `DO UPDATE`.

## Safe schema evolution

- Most DDL is transactional; every change still ships as a Flyway migration.
- `CREATE INDEX CONCURRENTLY` avoids blocking writes but cannot run in a transaction
  (`db-flyway-migrations`).
- Adding a column with a volatile default (`gen_random_uuid()`, `clock_timestamp()`, `random()`)
  rewrites the table; constants and `now()` do not.
- Drop constraints before columns. `CREATE OR REPLACE FUNCTION` with other arguments creates an
  overload.

## Generated columns

`… GENERATED ALWAYS AS (<expr>) STORED` for computed, indexable fields; PG18 adds `VIRTUAL`.

## Extensions

`pgcrypto` (digests; never password hashing — authentication belongs to the identity provider),
`pg_trgm` (fuzzy search, GIN for `LIKE '%x%'`), `citext`, `btree_gin`/`btree_gist`, `hstore`,
`postgis`, `pgvector`, `pgaudit`. `uuid-ossp` is not needed on PG18. Each needs a project decision.

## JSONB

- GIN index `USING GIN (col)` serves containment `@>`, key existence `?`, `?|`, `?&`;
  `jsonb_path_ops` is smaller/faster but supports containment only.
- Equality/range on one scalar field: extract it into a generated column with a B-tree index.
- Keep core relations in tables; constrain shape, e.g. `CHECK (jsonb_typeof(config) = 'object')`.

```sql
CREATE TABLE customer_profile (
  customer_id BIGINT PRIMARY KEY REFERENCES customer (id) ON DELETE CASCADE,
  attrs JSONB NOT NULL DEFAULT '{}' CHECK (jsonb_typeof(attrs) = 'object'),
  theme TEXT GENERATED ALWAYS AS (attrs->>'theme') STORED
);
CREATE INDEX idx_customer_profile_attrs ON customer_profile USING GIN (attrs);
```

## Other gotchas

- Sequences/identity have gaps (rollbacks, crashes) — normal, never "fix" them.
- Heap storage: no clustered PK; `CLUSTER` is a one-off reorganisation.
- MVCC: updates/deletes leave dead tuples for vacuum; avoid hot wide-row churn.
- No silent coercion: overflowing length/precision raises an error.

Source of this material: see the Provenance of `db-design-postgres-tables`.
