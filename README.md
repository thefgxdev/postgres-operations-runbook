# Postgres Operations Runbook

Postgres is enough for almost every product, until the day it is not, and that day is predictable. This runbook covers the operations that decide it: backups you have actually restored, migrations that do not lock, indexes that match the queries, connection pooling, the hot partition, and the queries to run when something is slow.

By [Felipe Guedes](https://fgxdev.com). Written from production incidents, most of them avoidable.

## Contents

- [`runbooks/backups.md`](runbooks/backups.md): base backups, WAL archiving, point-in-time recovery, and the restore drill that makes them real.
- [`runbooks/migrations.md`](runbooks/migrations.md): expand/migrate/contract, batched backfills, concurrent indexes, lock timeouts.
- [`runbooks/performance.md`](runbooks/performance.md): the queries to run first, `EXPLAIN (ANALYZE, BUFFERS)`, index hygiene, vacuum, connection pooling.
- [`runbooks/hot-partitions.md`](runbooks/hot-partitions.md): when one key gets all the traffic and sharding does not save you.
- [`runbooks/incident.md`](runbooks/incident.md): the first fifteen minutes when the database is the suspect.

## The five things to do this week if you have not

1. Restore last night's backup to a scratch instance and time it.
2. Set `statement_timeout` and `lock_timeout` for the application role.
3. Put a pooler (PgBouncer or the cloud equivalent) between the app and the database, and size pools from the database side.
4. Turn on `pg_stat_statements` and look at the top ten by total time.
5. Check `pg_index.indisvalid` for invalid indexes left by failed concurrent builds.

## Em português

Runbook de operação do Postgres: backups realmente restaurados, migrações sem lock, índices que batem com as consultas, pooling de conexões, partições quentes e os primeiros quinze minutos de um incidente. Artigos em [fgxdev.com/pt/articles](https://fgxdev.com/pt/articles/tag/data/).

## License

Apache-2.0. Copyright (c) 2026 Felipe Guedes (fgxdev.com). Redistributions must keep the NOTICE file and mark any changes.
