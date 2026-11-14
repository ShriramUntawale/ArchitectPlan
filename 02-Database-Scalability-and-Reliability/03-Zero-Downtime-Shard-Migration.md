# Chapter 3: Zero-Downtime PostgreSQL Shard Migration

## What you will learn

Adding a new shard changes where some records should live. A safe migration moves those records while the application continues operating. This chapter explains bulk copying, Change Data Capture (CDC), write fencing, checkpoints, ownership epochs, cutover, and migration-specific recovery.

Our running example is order `ORD-101`, which must move from Shard 2 to newly added Shard 6.

## 1. Why changing the hash ring is not enough

Before migration, `ORD-101` exists on Shard 2. The updated consistent hash ring says it belongs on Shard 6. The router must not immediately assume the record exists on Shard 6.

```text
Old ring: ORD-101 -> Shard 2 -> PENDING, version 5
New ring: ORD-101 -> Shard 6 -> record not yet copied
```

A direct switch could make a valid order appear missing. Worse, if the old and new shards both accept updates, their copies could disagree.

## 2. The complete strategy

```text
Plan migrating hash ranges
           |
Establish CDC + snapshot boundary
           |
Bulk copy existing records
           |
Apply changes captured during copying
           |
Fence old writes and drain in-flight writes
           |
Capture final source checkpoint
           |
Apply and verify changes through checkpoint
           |
Transfer authoritative ownership
           |
Allow new writes, monitor, and clean up
```

The old shard remains authoritative until the controlled cutover. Most unrelated orders continue operating throughout the migration.

## 3. Identify the keys that must move

Adding Shard 6 may take hash ranges from several existing shards because the ring contains virtual nodes. The migration planner compares the old and new ring ownership maps and produces a list of affected ranges.

| Order | Old owner | New owner | Move? |
| --- | --- | --- | --- |
| ORD-101 | Shard 2 | Shard 6 | Yes |
| ORD-102 | Shard 2 | Shard 2 | No |
| ORD-103 | Shard 3 | Shard 6 | Yes |

In production, track progress by hash range or bucket rather than maintaining a migration state for every order when millions of keys are involved.

## 4. Establish CDC before or at the snapshot boundary

A common mistake is copying millions of records first and only then starting to capture changes. Updates committed during the copy could be missed.

Instead, establish a consistent snapshot boundary and a change stream that covers writes occurring after that boundary. PostgreSQL logical decoding can read committed changes from the Write-Ahead Log (WAL). Tools such as Debezium can stream those changes to Kafka.

The bulk snapshot and CDC stream must be coordinated so no committed changes fall into a gap.

## 5. Copy records in batches

Suppose Shard 2 has 20 million orders, and four million must move to Shard 6. Copy manageable batches while Shard 2 continues handling production traffic.

```text
Shard 2                       Shard 6
  |-- batch 1 (10,000) ------->|
  |-- batch 2 (10,000) ------->|
  |-- batch 3 (10,000) ------->|
  |            ...             |
```

Track durable progress using a cursor or checkpoint. Use bounded concurrency to avoid overloading either database. The copy should be retryable without creating duplicate records.

## 6. Handle changes made during copying

At 10:00, the worker copies `ORD-101` at version 5. At 10:01, a customer confirms the order on Shard 2, creating version 6. At 10:02, the order is shipped, creating version 7.

| Time | Shard 2 | Shard 6 |
| --- | --- | --- |
| 10:00 | PENDING, v5 | PENDING, v5 |
| 10:01 | CONFIRMED, v6 | PENDING, v5 |
| 10:02 | SHIPPED, v7 | PENDING, v5 |
| After CDC applies changes | SHIPPED, v7 | SHIPPED, v7 |

CDC delivers committed changes for the migrating data to a destination applier. The applier must handle duplicates, ordering, retries, deletes, and related records correctly.

### Prevent an old bulk copy from overwriting newer data

A simplified versioned-row upsert can refuse older versions:

```sql
INSERT INTO orders (order_id, status, version)
VALUES ('ORD-101', 'PENDING', 5)
ON CONFLICT (order_id)
DO UPDATE SET
    status = EXCLUDED.status,
    version = EXCLUDED.version
WHERE orders.version < EXCLUDED.version;
```

If Shard 6 already contains version 7, this version 5 update changes nothing. For a real multi-table database, row versions alone are not enough. The migration must preserve transaction boundaries and correctly apply deletions and related changes.

## 7. Why CDC catch-up alone is not enough

Suppose Shard 6 catches up to Shard 2 at version 7. Immediately after the check, another request commits version 8 on Shard 2. If we switch routing before applying version 8, the new owner starts from stale data.

We need a final boundary after which the old shard cannot accept authoritative writes. This is the purpose of write fencing.

## 8. Understand write fencing

Fencing means revoking the old shard's permission to accept new authoritative writes for the migrating order or range. It does not mean stopping every write to the entire database.

For example, `ORD-101` is moving but `ORD-102` stays on Shard 2. We fence only the affected key or range. `ORD-102` remains writable.

```text
Before fence:
ORD-101 -> Shard 2 accepts writes
ORD-102 -> Shard 2 accepts writes

After fence:
ORD-101 -> Shard 2 rejects or delays new writes
ORD-102 -> Shard 2 still accepts writes
```

### Coordinate with in-flight transactions

Suppose Pod A already started updating `ORD-101`. The migration worker must not declare the old owner fenced while Pod A can still commit an old-owner write afterward.

One illustrative implementation uses a per-order ownership row lock. Every write transaction must lock and validate the ownership row before changing the order:

```sql
BEGIN;
SELECT writable FROM shard_ownership
WHERE order_id = 'ORD-101' FOR UPDATE;
-- Abort if writable is false.
UPDATE orders SET status = 'CONFIRMED', version = version + 1
WHERE order_id = 'ORD-101';
COMMIT;
```

The migration worker updates `writable` to false using the same row-lock coordination. If Pod A holds the lock, the worker waits until Pod A commits or rolls back. Once the fence transaction commits, later writes following the protocol cannot update the old owner.

This is a teaching example. A production system may fence whole hash ranges, use a strongly consistent ownership service, and require additional protections for stale connections and bypass paths.

## 9. Capture the final WAL checkpoint

After fencing and draining old writes, record a final source checkpoint. PostgreSQL WAL positions are called Log Sequence Numbers (LSNs).

```text
Final source boundary: LSN 5000
Destination durably applied: LSN 4900
Result: NOT READY

Final source boundary: LSN 5000
Destination durably applied: LSN 5000
Result: READY FOR VALIDATION
```

The destination must durably apply all relevant committed changes through the consistent final boundary. A connector reading an event is not the same as the destination committing it. LSN comparison must respect the snapshot, transaction, and applier checkpoint protocol.

Also validate row counts, versions, checksums where suitable, deletes, and related data. A matching checkpoint is necessary but does not prove every part of the migration is correct.

## 10. Transfer ownership with an epoch

An ownership epoch identifies the current generation of shard ownership. It differs from an order's data version.

| Field | Meaning |
| --- | --- |
| `order_version = 7` | The order has been updated through version 7 |
| `ownership_epoch = 2` | The second generation of ownership is authoritative |

Before cutover:

```text
Range R17 -> Shard 2, ownership epoch 1
```

After cutover:

```text
Range R17 -> Shard 6, ownership epoch 2
```

The routing metadata must be updated through a coordinated authoritative operation. The old write fence remains effective, and Shard 6 is enabled for authoritative writes only after the required data is ready. Requests may briefly wait or retry during this transition.

## 11. Handle a stale application pod

Pod A still caches `Shard 2, epoch 1`. It tries to update `ORD-101` after cutover.

```text
Pod A -> Shard 2 with epoch 1
Shard 2 -> Reject: stale ownership / write fenced
Pod A -> Refresh authoritative routing
Pod A -> Shard 6 with epoch 2
Shard 6 -> Validate and commit
```

The write boundary must enforce ownership. Checking only a local application cache is not sufficient. Bound retries by a deadline and make the operation idempotent so that retries do not duplicate business effects.

## 12. Migration failure scenarios

| Failure | Safe response |
| --- | --- |
| Bulk-copy worker crashes | Resume from durable batch checkpoint; old shard remains owner |
| CDC connector crashes | Resume from durable CDC checkpoint; keep required WAL available |
| Shard 6 fails before cutover | Keep Shard 2 authoritative; repair destination and resume |
| Worker crashes while fenced | Preserve durable migration state; resume catch-up or safely abort |
| Pod uses old routing after cutover | Reject old-owner write, refresh route, retry safely |
| Shard 6 fails after accepting new writes | Do not blindly route back to Shard 2; recover or reconcile newer committed data |

### Why post-cutover rollback is difficult

Suppose Shard 6 has accepted 500 new updates after becoming authoritative. Shard 2 does not contain them. Switching back to Shard 2 would hide or lose those committed changes from the application's point of view.

A safe reverse transfer requires recovering or synchronizing those updates, fencing the current owner, validating the destination, and transferring ownership again. If the new shard is unavailable, writes to affected keys may need to wait. General replication, disaster recovery, and automated failover belong in a separate future chapter.

## 13. Operational checklist

- [ ] Record old and new hash ring versions and affected key ranges.
- [ ] Establish a consistent snapshot and CDC boundary before copying.
- [ ] Make batch copying resumable and idempotent.
- [ ] Apply CDC changes without overwriting newer versions.
- [ ] Monitor copy progress, CDC lag, errors, and destination load.
- [ ] Fence old writes and drain in-flight transactions.
- [ ] Record and apply through the final source checkpoint.
- [ ] Validate migrated records and related data.
- [ ] Transfer ownership with an authoritative epoch change.
- [ ] Reject stale old-owner writes and support safe retries.
- [ ] Retain old data for a defined recovery period before cleanup.

## 14. The mental model to remember

Copy -> Catch up -> Fence -> Final catch-up -> Verify -> Switch -> Monitor.

Copying data is not the same as transferring ownership. CDC handles changes during copying. Fencing closes the final race. An ownership epoch prevents stale application pods from writing to the old owner.

## Practice question

Shard 2 is fenced, and Shard 6 has applied changes through the final checkpoint. Ownership switches to Shard 6. A stale pod attempts an update using epoch 1, while the current epoch is 2. What should happen, and why must the application refresh routing before retrying?
