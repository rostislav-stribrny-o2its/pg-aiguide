---
name: db-design-postgres-tables
description: |
  Use this skill for general PostgreSQL table design.

  **Trigger when user asks to:**
  - Design PostgreSQL tables, schemas, or data models when creating new tables and when modifying existing ones.
  - Choose data types, constraints, or indexes for PostgreSQL
  - Create user tables, order tables, reference tables, or JSONB schemas
  - Understand PostgreSQL best practices for normalization, constraints, or indexing
  - Design update-heavy, upsert-heavy, or OLTP-style tables


  **Keywords:** PostgreSQL schema, table design, data types, PRIMARY KEY, FOREIGN KEY, indexes, B-tree, GIN, JSONB, constraints, normalization, BCNF, lookup tables, identity columns, partitioning

  Binding table rules of this stack (standard columns, keys, constraints, naming) plus a non-binding PostgreSQL reference for data types, indexing, JSONB and partitioning.
license: Apache-2.0
metadata:
  author: tigerdata
---

# PostgreSQL Table Design

Target: **PostgreSQL 18**. The schema is changed only by Flyway migrations (`db-flyway-migrations`);
the application accesses it with Spring Data JDBC and `JdbcClient`.

This file holds the **binding rules** of the stack. General PostgreSQL knowledge (all data types,
index kinds, partitioning, extensions, JSONB, update/insert-heavy tables) is in
[reference.md](reference.md) and is **not binding** — use it to choose among options the binding
rules leave open.

## Binding rules

- **Normalize to at least BCNF, aim for 5NF** in the logical model; denormalize only for measured,
  high-ROI reads, and record why (ADR-00008 point 2).
- **Integrity in the database, business logic in the services.** Declarative constraints (PK, FK,
  UNIQUE, CHECK, EXCLUDE, NOT NULL) belong here; no triggers or procedures with business logic. A
  trigger may only guard integrity or immutability (example: an append-only table) (ADR-00008
  point 3).
- **Primary key** on every entity table: `BIGINT GENERATED ALWAYS AS IDENTITY (MAXVALUE
  9007199254740991)` — the bound 2^53−1 keeps every ID exact as a JSON number. **Never `UUID` as a
  primary key**; an opaque public identifier, when really needed, is a separate `UNIQUE` column.
- **Enumerated values are lookup tables** with a foreign key, rows shipped in `V` migrations —
  not `CREATE TYPE … AS ENUM`, not a free-text column with `CHECK (… IN (…))`. A lookup table has a
  small explicit key (`smallint`, assigned in the migration) and a unique `code` (ADR-00008 point 8).
- **Types:** `TIMESTAMPTZ` for instants (never `timestamp` without zone, `timetz` or a precision
  like `timestamptz(0)`; the one exception is ShedLock's own table, `be-scheduling-shedlock`), `DATE`
  for dates, `NUMERIC(p,s)` for money and every exact quantity (never float, never `money`),
  `BIGINT` for IDs, `INTEGER` where the range fits, `BOOLEAN NOT NULL`, identity never `serial`.
- **Strings with a limit:** `text` + `CHECK (length(col) <= n)`, or `varchar(n)` where the
  repository's existing migrations already use it — one style per schema, never mixed. The limit
  equals the `@Size(max = n)` of the API model (`be-api-contract-first`). No `char(n)`.
- **NOT NULL** wherever the value is required; **DEFAULT** for common values.
- **Row-level security** only by explicit project decision; authorization lives in the application.

### Standard columns (ADR-00008 points 10, 15)

```sql
id          BIGINT GENERATED ALWAYS AS IDENTITY (MAXVALUE 9007199254740991) PRIMARY KEY,
-- … business columns …
version     BIGINT NOT NULL DEFAULT 0,                 -- aggregate root only: optimistic locking (@Version)
created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
created_by  BIGINT REFERENCES persons (id) ON DELETE RESTRICT,   -- person id; NULL = system (job, migration)
updated_by  BIGINT REFERENCES persons (id) ON DELETE RESTRICT
```

- `id` + `version` on every aggregate root; `created_at`, `updated_at`, `created_by`, `updated_by`
  on every table holding user data. Lookup tables and pure junction tables have none of the four.
  `persons` is an *example* name of the application's person table.
- **Who fills them:** the service, on every write — times from the injected `Clock`, the person from
  `AppAuthentication.personId()` (`be-oauth2-resource-server`); jobs write `NULL` as the person.
  The database `DEFAULT now()` only covers an insert by plain SQL (migrations, test data). No
  trigger and no Spring Data auditing sets them.
- Deletion is hard; validity from–to only where the specification asks for it. Change history
  (who, when, what, from which value to which) is a separate decision of the specification.
- Index `created_by`/`updated_by` only when a query filters on them.

### Keys, constraints, indexes

- **Every FK states `ON DELETE`:** `RESTRICT` by default; `CASCADE` only for rows that belong to the
  parent (junction rows, parts of the same aggregate); `SET NULL` only for an optional reference
  whose loss is acceptable. `ON UPDATE` is never needed (identity keys do not change).
- **Index every FK column** (PostgreSQL does not), plus columns that are actually filtered, joined
  or sorted on.
- **UNIQUE:** plain `UNIQUE` when every column is `NOT NULL`. When a column of the key is nullable,
  decide: `UNIQUE NULLS NOT DISTINCT` if at most one `NULL` row is allowed, plain `UNIQUE` if many
  are.
- **CHECK** passes `NULL`; combine with `NOT NULL` where the value is required.
- **EXCLUDE** (GiST) for non-overlap rules, e.g. `EXCLUDE USING gist (room_id WITH =, period WITH &&)`.

### Naming

- `snake_case`, unquoted. Primary key `id`; foreign key `<referenced table, singular>_id`.
- Table names: plural (`persons`, `roles`, `integration_runs`) across all schemas — one convention
  everywhere. Exceptions: uncountable nouns (`integration_mirror`, `integration_staging`) and
  library-defined names (`shedlock`). Avoid reserved words (`user`, `order`).
- Constraint and index names: `idx_<table>_<columns>`, `uq_<table>_<columns>`,
  `fk_<table>_<column>`, `ck_<table>_<rule>`; an unnamed `PRIMARY KEY` / inline `UNIQUE` / inline
  `REFERENCES` keeps PostgreSQL's generated name.

## Example (the one example set of the stack's database skills)

```sql
-- V00002__create_order_status.sql — lookup table, reference data in the migration
CREATE TABLE order_status (
    id    SMALLINT PRIMARY KEY,                           -- explicit, assigned here
    code  TEXT NOT NULL UNIQUE CHECK (length(code) <= 30),
    name  TEXT NOT NULL
);
INSERT INTO order_status (id, code, name) VALUES
    (1, 'PENDING', 'Pending'), (2, 'CONFIRMED', 'Confirmed'), (3, 'CANCELLED', 'Cancelled');

-- V00003__create_customer.sql
CREATE TABLE customer (
    id          BIGINT GENERATED ALWAYS AS IDENTITY (MAXVALUE 9007199254740991) PRIMARY KEY,
    name        TEXT NOT NULL CHECK (length(name) <= 200),
    email       TEXT NOT NULL CHECK (length(email) <= 254),
    version     BIGINT NOT NULL DEFAULT 0,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by  BIGINT REFERENCES persons (id) ON DELETE RESTRICT,
    updated_by  BIGINT REFERENCES persons (id) ON DELETE RESTRICT
);
CREATE UNIQUE INDEX uq_customer_lower_email ON customer (LOWER(email));

-- V00004__create_customer_order.sql
CREATE TABLE customer_order (
    id               BIGINT GENERATED ALWAYS AS IDENTITY (MAXVALUE 9007199254740991) PRIMARY KEY,
    customer_id      BIGINT NOT NULL REFERENCES customer (id) ON DELETE RESTRICT,
    order_status_id  SMALLINT NOT NULL DEFAULT 1 REFERENCES order_status (id) ON DELETE RESTRICT,
    total_amount     NUMERIC(19,2) NOT NULL CHECK (total_amount >= 0),
    version          BIGINT NOT NULL DEFAULT 0,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by       BIGINT REFERENCES persons (id) ON DELETE RESTRICT,
    updated_by       BIGINT REFERENCES persons (id) ON DELETE RESTRICT
);
CREATE INDEX idx_customer_order_customer_id ON customer_order (customer_id);
CREATE INDEX idx_customer_order_order_status_id ON customer_order (order_status_id);
CREATE INDEX idx_customer_order_created_at ON customer_order (created_at);

-- V00005__create_customer_note.sql — a child row without its own natural key
CREATE TABLE customer_note (
    id          BIGINT GENERATED ALWAYS AS IDENTITY (MAXVALUE 9007199254740991) PRIMARY KEY,
    customer_id BIGINT NOT NULL REFERENCES customer (id) ON DELETE CASCADE,  -- notes die with the customer
    note        TEXT NOT NULL CHECK (length(note) <= 500),
    version     BIGINT NOT NULL DEFAULT 0,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by  BIGINT REFERENCES persons (id) ON DELETE RESTRICT,
    updated_by  BIGINT REFERENCES persons (id) ON DELETE RESTRICT
);
CREATE INDEX idx_customer_note_customer_id ON customer_note (customer_id);
```

## Gotchas

- Agent leaves out `created_by`/`updated_by` or `version` — standard columns above
- Agent fills `updated_at`/`updated_by` with a trigger or Spring Data auditing — the service sets them
- Agent writes a FK without `ON DELETE` — `RESTRICT` unless the child belongs to the parent
- Agent forgets the index on a FK column — PostgreSQL does not create it
- Agent uses `UUID` as primary key or `serial` — bounded `BIGINT` identity
- Agent creates `ENUM` types or `CHECK (status IN (…))` — lookup table with FK
- Agent uses `timestamp` without zone, `float` for money or `varchar` in a `text` schema — binding types
- Agent mixes naming conventions inside one schema — table names are plural (`persons`, `roles`, `integration_runs`; exceptions: `integration_mirror`, `integration_staging`, `shedlock`) and FKs are `<referenced table, singular>_id`
- Agent treats partitioning, extensions or RLS in reference.md as defaults — only on decision

## Provenance

Modified for the Elysium Java base (PostgreSQL 18, Flyway, Spring Data JDBC) — this file has been changed from the original.
Source: [timescale/pg-aiguide](https://github.com/timescale/pg-aiguide), `skills/design-postgres-tables`,
Apache License 2.0 (see the repository `LICENSE` and `NOTICE`: Copyright 2025 Timescale, Inc., d/b/a Tiger Data).
Changes: bounded `bigint` identities (2^53−1) instead of UUID keys, BCNF/5NF, lookup tables instead of ENUM/CHECK lists,
no business logic or password hashing in the database, RLS only by explicit decision, outdated partition FK note corrected,
TimescaleDB references removed, `NUMERIC` for exact quantities; split into binding rules (this file) and a
non-binding reference (`reference.md`), standard columns, `ON DELETE` and naming rules, one example set.
