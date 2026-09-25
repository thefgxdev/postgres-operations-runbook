# Performance: what to run first

## The top offenders

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

SELECT round(total_exec_time::numeric, 0) AS total_ms, calls,
       round(mean_exec_time::numeric, 1) AS mean_ms, left(query, 90) AS query
FROM pg_stat_statements
ORDER BY total_exec_time DESC LIMIT 15;
```

Total time, not mean, finds the query that actually costs you: a 5 ms query called a million times beats a 2 s query called twice.

## Explain the suspect

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) <query>;
```

Read for: `Seq Scan` on a large table, `Rows Removed by Filter` in the thousands, `Sort` spilling to disk (`external merge`), nested loops with a large outer side, `Buffers: shared read` far larger than `hit`.

## Index hygiene

```sql
-- unused indexes (after the stats have accumulated for a while)
SELECT indexrelid::regclass, idx_scan, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes WHERE idx_scan = 0 ORDER BY pg_relation_size(indexrelid) DESC;

-- duplicate indexes: same columns, same order
SELECT indrelid::regclass, array_agg(indexrelid::regclass), indkey
FROM pg_index GROUP BY indrelid, indkey HAVING count(*) > 1;
```

Rules: index columns in the order the query filters and sorts; composite indexes start with the most selective equality column; partial indexes for hot subsets (`WHERE status = 'pending'`); every foreign key column indexed.

## Vacuum and bloat

- `autovacuum` on, with more aggressive settings for hot tables (`autovacuum_vacuum_scale_factor = 0.02`).
- Long-running transactions block vacuum from reclaiming; find them with `pg_stat_activity WHERE state <> 'idle' ORDER BY xact_start`.
- Bloat estimate via `pgstattuple` on suspect tables; `pg_repack` to fix without locking.

## Connections

- Web instances × pool size must stay under a few hundred total. Use PgBouncer in transaction mode; size the pool from the database's `max_connections` and CPU count, not from the app's convenience.
- `idle_in_transaction_session_timeout` set (for example `60s`) so abandoned transactions do not hold locks.

## Timeouts

- `statement_timeout` for the application role (seconds, not minutes).
- Reports and batch jobs use a separate role with a larger timeout and a lower priority replica when possible.

## Checklist

- [ ] `pg_stat_statements` enabled and reviewed monthly.
- [ ] Top queries explained with `ANALYZE, BUFFERS`.
- [ ] No unused or duplicate indexes; every FK indexed.
- [ ] Autovacuum tuned for hot tables; no transactions older than minutes.
- [ ] Pooler in place; timeouts set per role.
