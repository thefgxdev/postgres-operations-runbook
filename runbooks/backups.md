# Backups

A backup that was never restored is a hope, not a backup.

## What to have

- **Base backup** daily (`pg_basebackup` or the managed equivalent), encrypted, stored in a different account or region from the database.
- **WAL archiving** continuous, so you can recover to any point in time, not only to last night.
- **Logical dump** (`pg_dump -Fc`) weekly for portability and for restoring a single table.
- **Retention** at least 30 days, longer if a corruption could go unnoticed longer (silent data bugs often do).

## The restore drill (monthly)

1. Restore the latest base backup plus WAL to a scratch instance.
2. Recover to a timestamp 10 minutes before "now".
3. Run the application's smoke tests against it.
4. Record: time to restore, size, any surprise.
5. Delete the scratch instance.

The recorded time is your real recovery time objective. If it is longer than the business accepts, fix that before the incident does.

## Point-in-time recovery, the short version

```
# restore base backup into $PGDATA, then in postgresql.conf / recovery settings:
restore_command = 'cp /archive/%f %p'
recovery_target_time = '2026-09-25 14:30:00'
recovery_target_action = 'promote'
```

Start the server; it replays WAL up to the target and promotes. Managed providers expose this as a button; still rehearse it.

## Restoring one table

From a logical dump: `pg_restore -t orders -d scratch dump.fc`, then copy the rows you need across. From PITR: restore the whole cluster to a scratch instance and `COPY` the table out.

## Alerts

- Backup job failed or did not run.
- WAL archive lag above a few minutes.
- Backup size changed by more than 30 % day over day (a silent truncate looks like this).
- Restore drill not run in 35 days.

## Checklist

- [ ] Base backup daily, off-site, encrypted.
- [ ] WAL archiving with lag alert.
- [ ] Restore drill in the last month, time recorded.
- [ ] Retention covers the longest plausible undetected corruption.
- [ ] Someone other than the person who set it up has done a restore.
