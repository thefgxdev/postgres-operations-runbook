# Migrations without downtime

## Lock timeouts first

Set them for every migration session so a migration that cannot get a lock fails fast instead of queueing behind a long query and blocking every write:

```sql
SET lock_timeout = '3s';
SET statement_timeout = '30s';   -- raise for known-long batched steps
```

## Operations and what they lock

| Operation | Safe? | Notes |
|---|---|---|
| `ADD COLUMN` nullable, no default | Yes | Instant metadata change |
| `ADD COLUMN ... DEFAULT x` | Yes on Postgres 11+ | Default stored in catalog, not rewritten |
| `ADD COLUMN NOT NULL` without default | No | Rewrites; add nullable, backfill, then `SET NOT NULL` with a `CHECK ... NOT VALID` then `VALIDATE` |
| `CREATE INDEX` | No | Locks writes; use `CONCURRENTLY` |
| `CREATE INDEX CONCURRENTLY` | Yes | Cannot run in a transaction; can leave an invalid index on failure |
| `ALTER COLUMN TYPE` | Usually no | Rewrites; add new column, backfill, swap |
| `DROP COLUMN` | Yes | Metadata only, space reclaimed later |
| `ADD FOREIGN KEY` | No | Use `NOT VALID` then `VALIDATE CONSTRAINT` |
| `RENAME` | Yes, but | Application must handle both names during rollout |

## Backfills

```sql
-- batched, resumable, rate-limited
UPDATE orders SET total_cents = total * 100
WHERE id IN (SELECT id FROM orders WHERE total_cents IS NULL ORDER BY id LIMIT 5000);
-- sleep 100 ms, repeat until 0 rows
```

- Batches of a few thousand rows.
- Idempotent: the `WHERE` selects only unmigrated rows.
- Progress observable: count of remaining rows in a dashboard.
- Verification before switching reads: counts, checksums, sampled diffs.

## Expand, migrate, contract

1. Expand: add the new structure; deploy code that writes both.
2. Migrate: backfill; switch reads behind a flag; verify.
3. Contract: after a full business cycle, remove the old structure. Write this ticket during step 1.

## Invalid indexes

```sql
SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid;
```

Drop and rebuild them. They take space and are not used.

## Checklist

- [ ] `lock_timeout` set in the migration session.
- [ ] Every operation checked against the table above.
- [ ] Backfills batched, idempotent, observable.
- [ ] Rollback rehearsed on a restored backup.
- [ ] Contract step ticketed with a date.
