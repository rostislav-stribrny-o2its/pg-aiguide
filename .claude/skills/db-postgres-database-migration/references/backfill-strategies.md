# Backfill Strategies

Backfilling (updating existing rows to populate a new column) on large tables must be done in batches to avoid long-running transactions, excessive locking, and WAL bloat.

**Elysium base:** a Flyway migration file is one transaction, so batched backfills do not run inside a V migration. Run them as a restartable, idempotent application job (for example a `@Scheduled` task with ShedLock, one `JdbcClient` batch per transaction) or as a controlled script step of the release, between the migration that adds the column and the migration that adds the constraint. The SQL below shows the batching logic; the progress table itself is created by a V migration.

## Batch by Primary Key

```sql
-- Backfill in chunks of 10,000 rows
-- Run this repeatedly until 0 rows affected
WITH batch AS (
    SELECT id FROM orders
    WHERE amount_new IS NULL
    ORDER BY id
    LIMIT 10000
    FOR UPDATE SKIP LOCKED
)
UPDATE orders
SET amount_new = amount::NUMERIC(12,2)
WHERE id IN (SELECT id FROM batch);
```

## Batch with Progress Tracking

```sql
-- Create a tracking table to resume if interrupted
CREATE TABLE migration_progress (
    migration_name TEXT PRIMARY KEY,
    last_processed_id BIGINT NOT NULL DEFAULT 0,
    rows_updated BIGINT NOT NULL DEFAULT 0,
    started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

INSERT INTO migration_progress (migration_name) VALUES ('backfill_amount_new');

-- Run in a loop from application code, or as this block OUTSIDE any transaction block
-- (psql with autocommit; COMMIT inside DO is rejected in a transaction, e.g. in Flyway):
DO $$
DECLARE
    v_batch_size CONSTANT INTEGER := 10000;
    v_last_id BIGINT;
    v_max_id BIGINT;
    v_rows INTEGER;
BEGIN
    SELECT last_processed_id INTO v_last_id
    FROM migration_progress WHERE migration_name = 'backfill_amount_new';
    SELECT coalesce(max(id), 0) INTO v_max_id FROM orders;

    -- Walk the whole id range: an empty window (identity gaps, rows already filled)
    -- must not stop the loop early
    WHILE v_last_id < v_max_id LOOP
        UPDATE orders
        SET amount_new = amount::NUMERIC(12,2)
        WHERE id > v_last_id AND id <= v_last_id + v_batch_size
          AND amount_new IS NULL;

        GET DIAGNOSTICS v_rows = ROW_COUNT;

        v_last_id := v_last_id + v_batch_size;

        UPDATE migration_progress
        SET last_processed_id = v_last_id,
            rows_updated = rows_updated + v_rows,
            updated_at = now()
        WHERE migration_name = 'backfill_amount_new';

        COMMIT;
        -- Yields to other transactions between batches
        PERFORM pg_sleep(0.1);
    END LOOP;
END;
$$;
```

## Backfill Considerations

- **Batch size:** Start with 10,000. Increase if each batch completes in under 1 second; decrease if it causes lock contention.
- **Sleep between batches:** 50–200ms gives other queries room. Tune based on your write load.
- **Monitor replication lag:** If you have replicas, check that the backfill doesn't cause them to fall behind.
- **VACUUM:** Run `VACUUM` (not `VACUUM FULL`) after a large backfill to reclaim dead tuple space without locking the table.

---

Modified for the Elysium Java base — this file has been changed from the original (Flyway context, loop exit over the full id range).
Source: [timescale/pg-aiguide](https://github.com/timescale/pg-aiguide), `skills/postgres-database-migration/references/backfill-strategies.md`, Apache License 2.0.
