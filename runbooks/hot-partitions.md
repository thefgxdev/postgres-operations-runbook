# Hot partitions

## The problem

One tenant, one product, one counter, one "latest" row receives most of the traffic. Sharding by that key does not help: the hot key lands on one shard and that shard is now the database. The fix is in the data model and the write pattern, not in adding machines.

## Recognise it

- Lock waits concentrated on a handful of rows (`pg_locks` joined with `pg_stat_activity`).
- One tenant's p99 much worse than the median tenant.
- A counter row (`views`, `balance`, `stock`) updated thousands of times per second.

## Patterns

**Counters: shard the counter, not the table.**
Keep N rows per counter (`counter_id, slot`), update a random slot, sum on read. Or append events and aggregate periodically.

**Latest row: stop updating it.**
Append events; the "current state" is a materialised view refreshed on a schedule, or a cached projection updated by a consumer.

**Sequential ids on a hot insert path.**
B-tree right-edge contention on monotonically increasing keys is usually fine in Postgres, but if a single table takes tens of thousands of inserts per second, partition by time and let each partition carry its own index.

**Per-tenant noisy neighbours.**
Rate limits and queues per tenant; bulk work on a separate queue; a separate replica for reports. Isolation is a product decision, not only a database one.

**Read-heavy hot keys.**
Cache them. A hot key by definition is small and read often; a cache with a short TTL removes most of the load. Invalidate on write through the same tag mechanism as everything else.

## What sharding is for

Sharding helps when load is spread across many keys and one machine cannot hold the total. It does not help when one key is the load. Fix the hot key first; you may not need to shard at all.

## Checklist

- [ ] Lock wait metrics per table and per tenant.
- [ ] No row updated more than a few hundred times per second; counters sharded or event-sourced.
- [ ] Per-tenant rate limits and queues.
- [ ] Hot read keys cached with explicit invalidation.
