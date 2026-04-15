# Database Migration Rules

Apply these rules whenever editing files under migrations/ or when asked to create/modify Alembic migrations.

## Cardinal Rules — Read Before Every Migration

1. NEVER auto-apply migrations — always show the generated SQL first
2. EVERY migration must have a working downgrade() function — no exceptions
3. Column drops and table drops require a two-phase migration strategy (see below)
4. ALWAYS review Alembic auto-generated output — it misses partial indexes, CHECK constraints, and custom types

## Generating Migrations

Always use the async engine config:
```bash
uv run alembic revision --autogenerate -m "TICKET-short-description"
```

After generating, IMMEDIATELY:
1. Review the generated file for correctness
2. Run `uv run alembic upgrade head --sql` to preview the actual SQL — share this in the PR
3. Check that downgrade() is complete and reversible

## Naming Conventions

Migration message format: `TICKET-short-description`
Examples:
- `FEAT-123-add-user-preferences-table`
- `FIX-456-add-index-on-events-created-at`
- `INFRA-789-backfill-tenant-id-column`

## Schema Change Playbook

### Safe changes (can deploy in one migration)
- Adding a nullable column
- Adding a new table
- Adding an index (use CONCURRENTLY — see below)
- Adding a foreign key on a new table
- Widening a varchar (e.g., 50 → 255)

### Unsafe changes (require multi-step migration)

**Renaming a column — 3 steps:**
1. Migration 1: Add new column, copy data with trigger or backfill
2. Code deploy: Write to both columns, read from old
3. Migration 2: Backfill remaining rows, switch reads to new column
4. Migration 3: Drop old column (after confirming no reads)

**Dropping a column — 2 steps:**
1. Code deploy: Remove all reads/writes to column
2. Migration: `op.drop_column(...)` — safe once no code references it

**Changing column type — treat as drop + add (3 steps above)**

**Adding a NOT NULL column — 2 steps:**
1. Migration 1: Add as nullable, backfill default value
2. Migration 2: Add NOT NULL constraint

### Index Creation — Always CONCURRENT

Alembic generates blocking CREATE INDEX by default. Override for all production tables:

```python
def upgrade() -> None:
    # CORRECT — non-blocking
    op.execute(
        "CREATE INDEX CONCURRENTLY IF NOT EXISTS "
        "ix_events_created_at ON events (created_at)"
    )

def downgrade() -> None:
    op.execute("DROP INDEX CONCURRENTLY IF EXISTS ix_events_created_at")
```

Do NOT use `op.create_index()` directly on tables with production traffic — it takes a table lock.

## downgrade() Must Work

Every migration must have a complete downgrade(). If truly irreversible (e.g., lossy data transform),
include a comment explaining why and raise NotImplementedError with a message:

```python
def downgrade() -> None:
    # This migration backfilled data that cannot be reversed without data loss.
    # If rollback is needed, restore from backup taken before upgrade.
    raise NotImplementedError(
        "Migration FEAT-123 is not reversible. See runbook: "
        "https://wiki.example.com/runbooks/FEAT-123-rollback"
    )
```

Never leave downgrade() as `pass` silently.

## Pre-Migration Checklist (add as comment to migration file)

```python
"""
FEAT-123-add-user-preferences-table

Pre-migration checklist:
[ ] Reviewed generated SQL with: alembic upgrade head --sql
[ ] downgrade() tested on staging
[ ] No table-locking operations on high-traffic tables (>1M rows)
[ ] CONCURRENT indexes used where applicable
[ ] Backfill script ready if column has NOT NULL constraint
[ ] Snapshot / backup confirmed before running on prod
[ ] Rollback tested: alembic downgrade -1

Risk level: LOW / MEDIUM / HIGH
"""
```

## Large Table Migrations

For tables with >1M rows, any structural change is HIGH RISK:
- Use CONCURRENTLY for indexes (mandatory)
- Column additions: use nullable + separate backfill job (not inline in migration)
- Set `lock_timeout` and `statement_timeout` at session level in migration
- Run during low-traffic window (off-peak hours) — document this in PR

```python
def upgrade() -> None:
    # Large table (events, ~50M rows) — setting timeouts to prevent lock pile-up
    op.execute("SET lock_timeout = '5s'")
    op.execute("SET statement_timeout = '30s'")
    op.add_column("events", sa.Column("tenant_id", sa.UUID, nullable=True))
```

## What Alembic Auto-generate Misses — Always Check

- **Partial indexes**: `CREATE INDEX ... WHERE condition` — add manually
- **CHECK constraints**: defined in SQLAlchemy via `CheckConstraint` but often missed
- **Custom PostgreSQL types**: enums, domains
- **Sequences** shared across tables
- **Triggers and functions**
- **Table inheritance**

After auto-generate, search the diff for these patterns and add manually if needed.

## Staging Gate

Migrations MUST be verified on staging before prod:
1. `kubectl exec -n staging deploy/SERVICE_NAME -- alembic upgrade head`
2. Confirm application starts successfully
3. Run smoke tests
4. Document result in PR description

No migration goes to prod without a staging gate. This is not optional.
