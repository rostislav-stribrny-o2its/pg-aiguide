# Complete Migration Example

End-to-end example: adding an order status (lookup table + foreign key column) to a large table,
as Flyway migrations in the Elysium Java base (PostgreSQL 18).

## Step 1: Plan and Document

```
Migration: Add order status to orders
- Table: orders (~5M rows, 2 GB)
- Change: lookup table order_status; column order_status_id smallint NOT NULL DEFAULT 1,
          FK to order_status (NOT VALID, then VALIDATE), partial index
- Files:  V00041__create_order_status.sql
          V00042__add_order_status_id_to_orders.sql
          V00043__validate_orders_order_status_fk.sql
          V00044__index_orders_active.sql            (CONCURRENTLY, alone in its file)
- Risk: Low (no rewrite: constant default on PG 11+; FK validated without blocking writes)
- Recovery: forward only — a follow-up migration drops the index, constraint and column;
            no data is destroyed, so no restore is needed
- Estimated duration: < 1 second for DDL, ~1 minute for VALIDATE, ~5 minutes for index build
```

## Step 2: Test on a Copy of Production Data

Restore the latest backup (or a dump) into a separate test database, then run the release's
migrations on it with Flyway, exactly as the deployment will. CI has already migrated an empty
database and upgraded from the last release.

## Step 3: The Migration Files

```sql
-- V00041__create_order_status.sql — reference data ships with the migration
CREATE TABLE order_status (
    id smallint PRIMARY KEY,
    code text NOT NULL UNIQUE CHECK (length(code) <= 30),
    name text NOT NULL
);
INSERT INTO order_status (id, code, name) VALUES
    (1, 'PENDING', 'Pending'), (2, 'COMPLETED', 'Completed'), (3, 'CANCELLED', 'Cancelled');

-- V00042__add_order_status_id_to_orders.sql
SET LOCAL lock_timeout = '5s';           -- never queue behind a long query
-- Fast: constant default, no rewrite (PG 11+)
ALTER TABLE orders ADD COLUMN order_status_id smallint NOT NULL DEFAULT 1;
-- FK without scanning the table now; new rows are checked immediately
ALTER TABLE orders ADD CONSTRAINT orders_order_status_fk
    FOREIGN KEY (order_status_id) REFERENCES order_status (id) NOT VALID;

-- V00043__validate_orders_order_status_fk.sql — separate file, lighter lock
ALTER TABLE orders VALIDATE CONSTRAINT orders_order_status_fk;

-- V00044__index_orders_active.sql — alone in its file (non-transactional)
CREATE INDEX CONCURRENTLY orders_active_idx
    ON orders (order_status_id, created_at DESC)
    WHERE order_status_id = 1;
```

`spring.flyway.postgresql.transactional-lock=false` must be set, otherwise the
`CONCURRENTLY` build waits on Flyway's own lock (see `flyway-migrations`).

## Step 4: Validate on the Copy

```sql
-- Column exists with correct type
SELECT column_name, data_type, is_nullable, column_default
FROM information_schema.columns
WHERE table_name = 'orders' AND column_name = 'order_status_id';

-- All existing rows have the default
SELECT order_status_id, COUNT(*) FROM orders GROUP BY order_status_id;

-- Constraint validated, index valid
SELECT conname, convalidated FROM pg_constraint WHERE conname = 'orders_order_status_fk';
SELECT indexrelid::regclass, indisvalid
FROM pg_index
WHERE indrelid = 'orders'::regclass AND NOT indisvalid;

-- Index is used
EXPLAIN ANALYZE
SELECT * FROM orders WHERE order_status_id = 1 ORDER BY created_at DESC LIMIT 10;
```

## Step 5: Release

The same four files are released; Flyway applies them in the deployment. Afterwards run the
validation queries of Step 4 against production (read-only). If `CREATE INDEX CONCURRENTLY`
failed and left an INVALID index, drop it in a NEW migration (`DROP INDEX CONCURRENTLY`, alone
in its file) and re-create it in the next one — never edit the released file.

## Step 6: Clean Up

Drop the test database with the production copy.

---

Modified for the Elysium Java base — this file has been changed from the original (Flyway files, lookup table with FK,
forward-only recovery, copy of production data instead of provider forks).
Source: [timescale/pg-aiguide](https://github.com/timescale/pg-aiguide), `skills/postgres-database-migration/references/complete-example.md`, Apache License 2.0.
