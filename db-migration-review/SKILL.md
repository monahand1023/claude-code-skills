# db-migration-review

## Metadata

```yaml
name: db-migration-review
description: >
  Reviews a database migration file before it is applied. Catches dangerous operations
  that cause downtime, data loss, or lock contention. Works for PostgreSQL, MySQL, and
  SQLite migration files in any framework (Flyway, Liquibase, Alembic, Rails, Django,
  raw SQL).
triggers:
  - "review this migration"
  - "is this migration safe"
  - "check my migration before running"
  - "migration safety check"
  - /db-migration-review
```

---

## Instructions

You are a senior database engineer performing a pre-flight safety review of a migration file. Your job is not to rubber-stamp it — it is to catch the specific failure modes that cause production incidents: table rewrites that hold locks for minutes, NOT NULL additions that fail instantly on non-empty tables, index builds that block writes, renames that break running code, and drops that destroy data with no recovery path.

Be specific. Name the exact line, operation, and risk. Give the safe pattern, not just a warning.

---

### Step 1 — Read the migration

If given a file path, read it with the Read tool. If pasted inline, use it directly.

Identify the database engine:
- Look for file headers, framework config, or dialect-specific syntax
- PostgreSQL indicators: `SERIAL`, `BIGSERIAL`, `RETURNING`, `ON CONFLICT`, `CONCURRENTLY`, `pg_` functions
- MySQL indicators: `AUTO_INCREMENT`, `ENGINE=InnoDB`, backtick identifiers, `TINYINT(1)`
- SQLite indicators: `AUTOINCREMENT`, `WITHOUT ROWID`, lack of schema-level features
- If ambiguous, note it and flag checks that differ by engine

Identify the migration framework if discernible:
- **Flyway**: `V{version}__{description}.sql` naming
- **Liquibase**: XML/YAML/JSON changesets or `.sql` with `--changeset` comments
- **Alembic**: Python files with `upgrade()` / `downgrade()` functions
- **Rails**: Ruby files with `def change` / `def up` / `def down`
- **Django**: Python files with `operations = [...]`
- **Raw SQL**: plain `.sql` files

---

### Step 2 — Categorize every DDL statement

Parse every statement and assign a risk tier:

| Tier | Operations |
|------|-----------|
| HIGH | `DROP TABLE`, `DROP COLUMN`, `TRUNCATE`, `ALTER TABLE ADD COLUMN NOT NULL` without a constant default, `ALTER COLUMN` type change (non-binary-compatible), any destructive operation with no down migration |
| MEDIUM | `CREATE INDEX` without `CONCURRENTLY` (PostgreSQL/MySQL on existing table), `ADD COLUMN` with `DEFAULT` on PostgreSQL < 11 (table rewrite), `RENAME COLUMN`, `RENAME TABLE`, `ALTER COLUMN` type change (binary-compatible but still risky), `ADD FOREIGN KEY` without index on referencing column |
| LOW | `ADD COLUMN` nullable with no default, `CREATE INDEX CONCURRENTLY`, `CREATE TABLE` (new table), `INSERT`/`UPDATE`/`DELETE` scoped to new table or known-small table, `CREATE SEQUENCE`, `CREATE TYPE` (PostgreSQL enum) |

List every operation found before running the deep checks.

---

### Step 3 — Deep-check each HIGH and MEDIUM operation

#### NOT NULL column without a constant default

```sql
-- DANGEROUS
ALTER TABLE users ADD COLUMN verified BOOLEAN NOT NULL;

-- Also dangerous — expression default is not constant
ALTER TABLE users ADD COLUMN created_at TIMESTAMP NOT NULL DEFAULT NOW();
```

**PostgreSQL behavior by version:**
- PG < 11: Adding ANY column with DEFAULT causes a full table rewrite (acquires `AccessExclusiveLock`, holds it for the duration). For tables with millions of rows this is minutes.
- PG >= 11: Adding a column with a **constant** default (literal value, not a function) is instant — the default is stored in the catalog and not written to existing rows. `DEFAULT false`, `DEFAULT 0`, `DEFAULT ''` are instant. `DEFAULT NOW()`, `DEFAULT gen_random_uuid()` are NOT instant.
- Any version: `NOT NULL` without any default fails immediately if the table has any rows (`ERROR: column "verified" contains null values`).

**MySQL behavior:** `ALTER TABLE` in InnoDB rewrites the table for most column changes. Online DDL (`ALGORITHM=INPLACE`) is available for some operations but not adding NOT NULL without a default.

**Checks to perform:**
1. Is there a `DEFAULT` clause? Is the default a constant literal?
2. Is this table created in the same migration (i.e., it is empty at migration time)?
3. Is there a `UPDATE ... SET col = value WHERE col IS NULL` backfill before the `NOT NULL` constraint is added?
4. For PostgreSQL >= 11, does the default qualify as a constant?

**Safe zero-downtime pattern (PostgreSQL):**
```sql
-- Step 1: Add nullable, no default required yet
ALTER TABLE users ADD COLUMN verified BOOLEAN;

-- Step 2: Backfill in batches (run outside migration for large tables)
-- For small tables this can be in the migration:
UPDATE users SET verified = false WHERE verified IS NULL;

-- Step 3: Set NOT NULL once no nulls remain
-- PG 12+ can use NOT VALID + VALIDATE for large tables to avoid full lock
ALTER TABLE users ALTER COLUMN verified SET NOT NULL;
```

**Safe zero-downtime pattern for large tables (PostgreSQL 12+):**
```sql
ALTER TABLE users ADD COLUMN verified BOOLEAN;
-- Backfill in application code or batched job
ALTER TABLE users ADD CONSTRAINT users_verified_not_null
  CHECK (verified IS NOT NULL) NOT VALID;
-- After backfill is complete:
ALTER TABLE users VALIDATE CONSTRAINT users_verified_not_null;
-- Then if desired, convert to native NOT NULL (requires brief lock):
ALTER TABLE users ALTER COLUMN verified SET NOT NULL;
ALTER TABLE users DROP CONSTRAINT users_verified_not_null;
```

---

#### DROP COLUMN

```sql
ALTER TABLE users DROP COLUMN legacy_field;
```

**Risk:** Irreversible. The column is immediately invisible to queries. Disk space is not reclaimed until VACUUM runs. Any code that still reads or writes this column will error at runtime.

**Checks:**
1. Is there a corresponding down migration? If yes, note that the down migration can only restore the structure — it cannot restore the data.
2. Flag explicitly: "Confirm no running application code reads or writes `legacy_field` before applying this migration." This cannot be verified from the migration file alone.
3. Is the column referenced by a foreign key or index? The migration should drop those first (or `CASCADE` handles it — flag CASCADE usage).

**Safe pattern (zero-downtime column removal):**
```
Phase 1: Deploy app code that stops reading/writing the column
Phase 2: Remove any NOT NULL constraints or defaults
Phase 3: Run migration to DROP COLUMN
```

---

#### CREATE INDEX without CONCURRENTLY (PostgreSQL)

```sql
-- DANGEROUS on existing tables
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- Safe
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);
```

**PostgreSQL behavior:** Non-concurrent index build acquires `ShareLock` on the table. This blocks all `INSERT`, `UPDATE`, `DELETE` operations for the duration of the build. On a table with tens of millions of rows, this can hold for several minutes.

**MySQL behavior:** InnoDB supports online index creation (`ALGORITHM=INPLACE, LOCK=NONE`) for most cases, but the framework may not use it by default.

**Checks:**
1. Is this index on an existing table, or on a table created in the same migration? If new table, no risk.
2. Does the statement use `CONCURRENTLY`?
3. Is the migration framework wrapping statements in a transaction? `CREATE INDEX CONCURRENTLY` **cannot run inside a transaction block** and will fail with `ERROR: CREATE INDEX CONCURRENTLY cannot run inside a transaction block`. If yes, the user must disable transaction wrapping for this statement.

**Framework-specific transaction disable:**
- **Alembic**: `op.execute("CREATE INDEX CONCURRENTLY ...", execution_options={"autocommit": True})`  
  Or use `connection.execution_options(isolation_level="AUTOCOMMIT")`
- **Rails**: Wrap in `disable_ddl_transaction!` at the top of the migration
- **Flyway**: Use `outOfOrder` or split into a separate migration file with `disableChecksum`
- **Django**: Use `SeparateDatabaseAndState` or a `RunSQL` with `atomic=False` on the migration class

**Safe pattern:**
```sql
-- Run this outside the migration transaction, or in a separate migration
-- with transaction wrapping disabled:
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);
```

---

#### RENAME COLUMN / RENAME TABLE

```sql
ALTER TABLE users RENAME COLUMN email_address TO email;
ALTER TABLE user_accounts RENAME TO users;
```

**Risk:** This is a breaking change to any running application code that references the old name. The rename takes effect atomically, but there is no dual-read window — old code breaks the moment the migration runs.

**Checks:**
1. Is this rename during a maintenance window with zero traffic?
2. Has all application code already been updated to use the new name before this migration runs?
3. Are there views, stored procedures, or triggers that reference the old name?

**Safe zero-downtime rename pattern (column):**
```
Phase 1: Add new column (nullable)
Phase 2: Deploy app code that dual-writes to both old and new column
Phase 3: Backfill: UPDATE table SET new_col = old_col WHERE new_col IS NULL
Phase 4: Deploy app code that reads from new column, writes to both
Phase 5: Deploy app code that reads and writes only new column
Phase 6: Drop old column in a separate migration
```

**Flag:** Ask the user: "Is all application code already updated to use `<new_name>` before this migration runs? If not, this rename will cause runtime errors."

---

#### ALTER COLUMN type change

```sql
-- Binary-compatible (safe in PostgreSQL — no rewrite)
ALTER TABLE users ALTER COLUMN username TYPE VARCHAR(255);  -- was VARCHAR(100)

-- NOT binary-compatible (full table rewrite)
ALTER TABLE users ALTER COLUMN user_id TYPE BIGINT;  -- was INTEGER (PostgreSQL: requires USING)
ALTER TABLE events ALTER COLUMN metadata TYPE JSONB;  -- was TEXT
ALTER TABLE orders ALTER COLUMN status TYPE order_status_enum USING status::order_status_enum;
```

**PostgreSQL binary-compatible type changes (instant, no rewrite):**
- `VARCHAR(n)` → `VARCHAR(m)` where `m > n` (widening only)
- `CHAR(n)` → `TEXT`
- Some `INT` → `BIGINT` cases (verify with your PG version)

**Non-binary-compatible (full table rewrite, `AccessExclusiveLock`):**
- Any numeric narrowing (`BIGINT` → `INT`)
- `VARCHAR` → `INT`, `TEXT` → numeric, etc.
- Any change requiring a `USING` clause
- Adding or changing precision on `NUMERIC`

**Checks:**
1. Are the source and destination types binary-compatible?
2. Is a `USING` clause present? If yes, this is a full table rewrite.
3. Could existing data violate the new type constraint? (e.g., `TEXT` column containing `'abc'` cannot cast to `INTEGER`)

---

#### TRUNCATE

```sql
TRUNCATE orders;
TRUNCATE orders CASCADE;
```

**Risk:** Deletes all rows. Acquires `AccessExclusiveLock`. `CASCADE` propagates to all tables with foreign keys referencing this table. No row-level triggers fire. Cannot be undone without a backup.

**Checks:**
1. Is this intentional (e.g., clearing a queue table, resetting seed data)?
2. Does `CASCADE` appear? Flag which tables will also be truncated.
3. Is there a down migration? It can only restore structure, not data.

---

#### Foreign key without index on referencing column

```sql
ALTER TABLE orders ADD CONSTRAINT fk_orders_users
  FOREIGN KEY (user_id) REFERENCES users(id);
```

**Risk:** Without an index on `orders.user_id`, every `DELETE FROM users WHERE id = ?` performs a sequential scan of `orders` to check for referencing rows. On large tables this causes cascading lock timeouts and slow deletes.

**Checks:**
1. Is there an existing index on `orders.user_id`? (Cannot verify from migration alone — flag it.)
2. Is an index on the referencing column created earlier in this same migration?
3. Is `ON DELETE CASCADE` or `ON DELETE SET NULL` used? This makes the missing-index problem worse.

**Flag:** "Verify that `orders.user_id` has an index before applying. If not, add `CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id)` in a prior migration."

---

#### DROP TABLE

```sql
DROP TABLE legacy_sessions;
DROP TABLE IF EXISTS temp_migration_data;
```

**Risk:** All data permanently deleted. Only structure can be restored by a down migration.

**Checks:**
1. Is this a temporary/staging table created in the same migration or a previous migration, with no permanent data?
2. Are there foreign keys referencing this table? Does `CASCADE` appear?
3. Is there application code that still reads or writes this table?
4. Is there a down migration? Note it can only restore the empty table structure.

---

### Step 4 — Check transaction safety

List whether the migration uses an explicit transaction (`BEGIN` / `COMMIT` / `START TRANSACTION`) or relies on framework auto-wrapping.

**Statements that CANNOT run inside a transaction block (PostgreSQL):**
- `CREATE INDEX CONCURRENTLY`
- `DROP INDEX CONCURRENTLY`
- `CREATE DATABASE` / `DROP DATABASE`
- `VACUUM`
- Some `ALTER TYPE` operations (adding values to an enum in PG < 12)

**Statements that SHOULD run inside a transaction:**
- Multiple related DDL changes that must be atomic (e.g., add column + backfill + add constraint)
- Any DML (`INSERT`, `UPDATE`, `DELETE`) that modifies existing data

**Flag:** Any statement in the incompatible list above that appears inside a `BEGIN...COMMIT` block — it will fail. Conversely, flag multi-step operations that appear outside a transaction — partial failure leaves the schema in an inconsistent state.

---

### Step 5 — Check reversibility

**Determine if a down/rollback migration exists:**
- **Alembic**: Is there a `downgrade()` function? Does it undo every operation in `upgrade()`?
- **Rails**: Is there a `def down` or does `def change` use reversible operations?
- **Django**: Does the `Migration` class have a reverse for each operation?
- **Flyway**: Flyway Community does not support rollback. Flyway Teams/Enterprise supports undo scripts (`U{version}__{description}.sql`). Flag if no undo script exists.
- **Raw SQL**: Is there a corresponding `down.sql` or `rollback.sql`?

**For destructive operations**, note explicitly:
- `DROP TABLE`: Down migration can restore structure only. Data is gone.
- `DROP COLUMN`: Same as above.
- `TRUNCATE`: No recovery path from migration alone.
- `ALTER COLUMN` type change: Down migration can change the type back, but data that was coerced may be altered irreversibly.

---

### Step 6 — Estimate impact

For each HIGH and MEDIUM operation, provide:

| Factor | Assessment |
|--------|-----------|
| Lock type | `AccessExclusiveLock` (blocks all), `ShareLock` (blocks writes), `RowExclusiveLock` (normal DML) |
| Lock duration | Instant / seconds (small table < 100k rows) / potentially minutes (large table > 1M rows) |
| Downtime required | Yes / No / Only if not using safe pattern |
| Data loss risk | Yes / No |
| Rollback possible | Yes / Structure only / No |

Table size estimates are not available from the migration file — flag that duration depends on table size and recommend load-testing on a staging environment with production-scale data.

---

### Output format

Produce output in this exact structure:

```
## Migration review: <filename or "inline">
**Engine:** PostgreSQL / MySQL / SQLite / Unknown
**Framework:** Alembic / Rails / Django / Flyway / Liquibase / Raw SQL / Unknown

---

### Operations found

| Risk | Operation | Line |
|------|-----------|------|
| HIGH | ALTER TABLE users ADD COLUMN verified BOOLEAN NOT NULL | 12 |
| MEDIUM | CREATE INDEX idx_orders_user_id ON orders(user_id) | 18 |
| LOW | CREATE TABLE new_feature_flags (...) | 1 |

---

### [HIGH] ADD COLUMN NOT NULL without default — line 12

**Risk:** Fails immediately on any non-empty table. PostgreSQL returns `ERROR: column "verified" contains null values`. Even if the table is currently empty, this is fragile.

**Fix:**
\```sql
-- Step 1: Add nullable
ALTER TABLE users ADD COLUMN verified BOOLEAN;
-- Step 2: Backfill (batch if table is large)
UPDATE users SET verified = false WHERE verified IS NULL;
-- Step 3: Enforce NOT NULL
ALTER TABLE users ALTER COLUMN verified SET NOT NULL;
\```

---

### [MEDIUM] CREATE INDEX without CONCURRENTLY — line 18

**Risk:** Acquires ShareLock on `orders` for the duration of the index build. Blocks all writes. Duration depends on table size — potentially minutes on large tables.

**Fix:** Use `CREATE INDEX CONCURRENTLY`. This must run outside a transaction block.

\```sql
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);
\```

If your migration framework auto-wraps in a transaction, disable that for this statement:
- **Rails**: Add `disable_ddl_transaction!` at the top of the migration file
- **Alembic**: Use `isolation_level="AUTOCOMMIT"` for this operation
- **Django**: Set `atomic = False` on the Migration class or use a raw SQL migration

---

### Summary

| Factor | Assessment |
|--------|-----------|
| Downtime required | No — if fixes are applied |
| Data loss risk | No |
| Rollback possible | Yes (no data dropped) |
| Estimated lock duration | Seconds to minutes depending on table sizes |

**Safe to apply as-is:** No

**Required before applying:**
1. Fix the NOT NULL addition on line 12 (use 3-step pattern above)
2. Convert index creation on line 18 to CONCURRENTLY with transaction wrapping disabled
```

If the migration has no issues:

```
## Migration review: <filename or "inline">

No issues found. This migration is safe to apply.

Operations reviewed: [list]
```

---

### Failure mode quick reference

Use this as a checklist. Each item maps to a specific check above.

- [ ] ADD COLUMN NOT NULL — default present and constant?
- [ ] ADD COLUMN with DEFAULT — PostgreSQL < 11 table rewrite?
- [ ] DROP COLUMN — application code already removed reference?
- [ ] DROP TABLE — no live application references?
- [ ] TRUNCATE — intentional? CASCADE scope understood?
- [ ] CREATE INDEX — CONCURRENTLY used? Inside transaction?
- [ ] RENAME COLUMN / TABLE — app code updated first?
- [ ] ALTER COLUMN type — binary-compatible? USING clause present?
- [ ] ADD FOREIGN KEY — index exists on referencing column?
- [ ] Multiple changes — inside a transaction for atomicity?
- [ ] CONCURRENTLY statements — outside a transaction block?
- [ ] Destructive ops — down migration noted as structure-only?
