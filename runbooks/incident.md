# The first fifteen minutes

The database is the suspect. Do not restart it yet.

## Minute 0 to 2: is it the database?

```sql
SELECT count(*) FILTER (WHERE state = 'active') AS active,
       count(*) FILTER (WHERE wait_event_type = 'Lock') AS waiting_on_locks,
       count(*) AS total
FROM pg_stat_activity WHERE backend_type = 'client backend';
```

- Hundreds active and most waiting on locks: a blocker. Go to minute 2.
- Total near `max_connections`: connection storm. Go to minute 5.
- Few active, app still slow: it is probably not the database. Check the app, the network, the pooler.

## Minute 2 to 5: find the blocker

```sql
SELECT pid, now() - xact_start AS age, state, wait_event_type, left(query, 80)
FROM pg_stat_activity
WHERE pid IN (SELECT unnest(pg_blocking_pids(pid)) FROM pg_stat_activity)
ORDER BY age DESC;
```

The oldest blocking transaction is usually a migration, a long report, or an `idle in transaction` session from a crashed worker. `SELECT pg_cancel_backend(pid)` first; `pg_terminate_backend(pid)` if it does not stop. Write down what it was.

## Minute 5 to 10: connection storm

- Symptom: `FATAL: too many connections`, app timing out on connect.
- Cause: pools sized from the app side, or a deploy that doubled instances, or retries without backoff.
- Now: reduce app instances or pool size; terminate idle sessions older than N minutes; put the pooler in front if it is missing.
- After: pooler in transaction mode, pool sizes from the database side, `idle_in_transaction_session_timeout`.

## Minute 10 to 15: disk, memory, replication

- Disk full: WAL not archived, or a runaway temp file from a sort. `SELECT pg_size_pretty(pg_database_size(current_database()))`, check the WAL directory, check `temp_files` in `pg_stat_database`.
- Replica lag: `SELECT now() - pg_last_xact_replay_timestamp()` on the replica. If reads are served from a lagging replica, users see old data; route critical reads to the primary while you find the cause.
- Memory: `work_mem` × concurrent sorts can exceed RAM. Lower `work_mem`, find the query with the big sort.

## Before you restart anything

A restart clears the symptom and destroys the evidence. Capture `pg_stat_activity`, `pg_locks` and the top of `pg_stat_statements` to a file first.

## After

Postmortem within 48 hours: timeline, root cause, what would have caught it earlier, the change that prevents it. One owner per action item.
