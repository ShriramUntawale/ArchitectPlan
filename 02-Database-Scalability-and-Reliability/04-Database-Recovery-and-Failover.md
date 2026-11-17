# Chapter 04 - Database Recovery and Failover

## What you will learn

This chapter explains how a PostgreSQL-based order platform survives server crashes, network partitions, zone outages, accidental deletion, and backup failures. We use one continuing example: an order service storing `ORD-101` in PostgreSQL. The business requires zero loss of acknowledged orders (RPO target) and recovery within five minutes (RTO target). These are design targets, not guarantees automatically supplied by PostgreSQL.

By the end, you should be able to explain the difference between a replica and a backup, select a safe failover candidate, prevent split-brain, plan point-in-time recovery, and evaluate a disaster-recovery drill.

## 1. Primary and standby replicas

A PostgreSQL primary accepts writes. A standby receives changes from the primary, usually through Write-Ahead Log (WAL) streaming. A standby can serve some reads, but replication lag can make those reads stale.

```text
                Order API
                    |
                 Writes
                    v
              Primary P
               /     \
          WAL /       \ WAL
             v         v
         Standby A  Standby B
```

Example: the primary commits `ORD-101`. Standby A has received it, but Standby B still contains only `ORD-100`. If the primary fails, Standby B is not an acceptable promotion choice for a zero acknowledged-loss objective if the missing order cannot be recovered.

A healthy replica is not necessarily a sufficiently up-to-date replica. Low CPU usage is not a reason to prefer a standby that is missing acknowledged transactions.

## 2. WAL: the database's change history

WAL stands for Write-Ahead Log. PostgreSQL records changes in WAL before treating the corresponding transaction as durably committed. WAL supports crash recovery and replication.

A Log Sequence Number (LSN) identifies a position in a WAL stream. Standbys may have different positions for WAL received, WAL flushed to durable storage, and WAL replayed into visible database state.

On a standby, these queries are useful:

```sql
SELECT pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn();
```

The receive LSN does not alone prove that WAL is durably flushed. A failover controller must consider the actual durability policy, timeline, and standby state.

### What if the primary is unreachable?

You can inspect the WAL progress of surviving standbys without connecting to the failed primary. But the most advanced surviving standby may still be behind the primary's last acknowledged transaction if replication was asynchronous. You cannot reconstruct the primary's unknown final position by guessing.

## 3. Asynchronous and synchronous replication

### Asynchronous replication

```text
Client -> Primary: INSERT ORD-101
Primary -> Client: success
Primary -> Standby: WAL arrives later
```

The primary can acknowledge before a standby has durably received the transaction. This reduces write latency, but a primary failure may lose an acknowledged transaction from the surviving history.

### Synchronous replication

```text
Client -> Primary: INSERT ORD-101
Primary -> Standby: WAL
Standby -> Primary: required durable acknowledgement
Primary -> Client: success
```

With a properly configured synchronous standby and `synchronous_commit = on`, PostgreSQL waits for the required standby to flush WAL before acknowledging commit. This is different from waiting for the standby to replay the transaction for reads. The exact guarantee depends on `synchronous_standby_names`, commit settings, and which standbys are eligible.

If two standbys exist but only one must acknowledge, do not assume both have the latest order. Promoting the wrong one can still lose acknowledged data.

### What if both synchronous standbys disappear?

If the business insists on zero acknowledged-order loss across a primary failure, do not silently downgrade to asynchronous acknowledgements. Required synchronous commits may wait. The API should apply admission control and avoid exhausting connection pools. A timed-out request may have an uncertain commit outcome, so use idempotency keys and status checks before retrying.

The trade-off is clear: stronger durability can reduce write availability during replication failures.

## 4. Failover: replacing a failed primary

Failover changes which PostgreSQL instance is the writable primary of a shard group. It does not change which logical shard owns the orders.

```text
Before failure:                 After safe failover:
Shard 4 -> Primary P            Shard 4 -> Standby A promoted
             |                                |
          Standby A                        New primary
```

A safe failover procedure is:

1. Detect the primary failure or loss of authority.
1. Establish that the failover controller has valid coordination authority.
1. Reliably fence or demote the old primary so it cannot continue accepting writes.
1. Verify that the promotion candidate satisfies the acknowledged-data durability policy.
1. Promote the eligible standby.
1. Redirect database connections and reconnect the application.
1. Verify writes, order integrity, and application health.

If the old primary comes back, it must not automatically resume as a writable primary. It must be safely rejoined, often after reinitialization or rewind as appropriate.

## 5. Split-brain and fencing

Split-brain means two database instances both act as authoritative writers for the same data.

Example: the failover controller cannot reach Primary P, but P is still running and some application pods can reach it. Promoting Standby A without first removing P's write authority risks conflicting order histories.

Fencing means making an old owner unable to perform authoritative writes. Methods can include verified infrastructure power-off, storage fencing where relevant, or a correctly enforced HA demotion protocol. Removing a host from a load balancer alone may not close existing connections or alternate routes.

```text
Old primary unreachable
          |
          v
Confirm old write authority revoked
          |
          v
Verify candidate WAL and eligibility
          |
          v
Promote new primary
```

If infrastructure power-off cannot be confirmed, do not assume it succeeded. For a strict safety-first policy, postpone promotion until exclusive write authority is established.

### Why checking a lease before each write is not enough

A process might check that its lease is valid, freeze for 45 seconds, and resume after the lease expires. A new primary might already exist. This is a time-of-check versus time-of-use race. The write boundary must enforce leadership or the old node must be independently fenced. A cached timestamp check is not sufficient.

## 6. etcd quorum, leader leases, and Patroni

etcd is a distributed coordination store. A three-node etcd cluster needs a majority of two nodes for quorum.

```text
Network partition:

Side A: etcd 3 + old Primary P     -> 1 of 3, no quorum
Side B: etcd 1 + etcd 2 + standbys -> 2 of 3, quorum
```

A Patroni-style HA manager uses a distributed configuration store for leadership coordination. etcd does not itself inspect PostgreSQL WAL and decide which replica is newest. The HA manager combines coordination state with PostgreSQL eligibility and replication information.

The old primary must stop acting as leader if it cannot retain authority. The majority side can coordinate a promotion only after the old writer is safely excluded and a suitable standby is available.

A leader lease is temporary authority, not permanent ownership. Shorter leases may speed failure detection but increase sensitivity to pauses and network problems. Longer leases tolerate more delay but may lengthen failover.

### Why three Availability Zones matter

If etcd nodes 1 and 2 are both in Zone A and node 3 is in Zone B, losing Zone A removes the majority. Healthy standbys in Zone B cannot use that etcd cluster to coordinate automatic promotion.

Distributing one etcd node across each of three independent zones allows a single-zone failure to leave two surviving quorum members, assuming the network between them works. Database replication and fencing still require separate design.

## 7. RPO and RTO

RPO (Recovery Point Objective) describes acceptable data loss, usually expressed as time. Here the business interpretation is zero acknowledged orders lost.

RTO (Recovery Time Objective) describes the allowed time to restore the required service. Here it is five minutes.

A quick recovery can violate RPO. A lossless recovery can violate RTO. We must measure both.

Example failover drill:

| Operation | Time |
| --- | ---: |
| Failure detection | 40 seconds |
| Fence old primary | 30 seconds |
| Promote standby | 50 seconds |
| Application reconnect | 60 seconds |
| Health validation | 45 seconds |
| Total | 225 seconds |

The measured steps total 3 minutes 45 seconds, below the five-minute threshold. But if three acknowledged orders are missing, the drill fails data integrity and cannot be declared a successful recovery. The service's actual RTO may also remain unmet if correct service is not restored.

## 8. Database failover versus database resharding

These operations solve different problems.

| Failover | Resharding |
| --- | --- |
| Replace a failed primary | Change where order IDs belong |
| Keep logical shard ownership | Transfer logical shard ownership |
| Promote that shard's standby | Bulk copy, CDC, fence, cut over |
| Restore availability | Redistribute data or capacity |

If Shard 4's primary fails, promote Shard 4's eligible standby. Do not automatically scatter Shard 4's orders across Shards 1, 2, and 3. Those shards do not necessarily contain Shard 4's data.

### N+1 capacity is not the same as availability

If a primary sustains 250 orders/sec and the target utilization is 70%, planned capacity per group is:

$$
250 \times 0.70 = 175\text{ orders/sec}
$$

For a peak of 400 orders/sec, the mathematical minimum is:

$$
\left\lceil 400/175 \right\rceil = 3\text{ groups}
$$

Four groups provide 700 orders/sec of aggregate planned capacity, and three surviving groups would have 525 orders/sec. But spare aggregate capacity cannot substitute for the missing shard's data. A standby for each shard group addresses primary failure without moving its keys.

## 9. Entire Availability Zone failure

Suppose four primaries run in Zone A and their synchronous standbys run in Zone B. Zone A fails completely.

```text
Zone A (failed)             Zone B (healthy)
Shard 1 primary  -X->       Shard 1 standby
Shard 2 primary  -X->       Shard 2 standby
Shard 3 primary  -X->       Shard 3 standby
Shard 4 primary  -X->       Shard 4 standby
```

Recovery requires a surviving coordination quorum, old-primary fencing, verified durable WAL on each candidate, and safe promotion. Independent shard groups can be promoted concurrently if the HA controller and infrastructure support it. Application routing then changes to the new primary endpoints, while shard ownership remains the same.

Synchronous replication does not by itself prove that every standby is current. Verify the actual acknowledgement policy and candidate eligibility for each shard group.

## 10. Point-in-Time Recovery (PITR)

Failover handles an unavailable database server. PITR handles a different problem: returning a database to a previous state after logical corruption or accidental deletion.

Example:

| Time | Event |
| --- | --- |
| 10:00 | Base backup |
| 10:05 | 1,000 orders created |
| 10:10 | 2,000 more orders created |
| 10:15 | `DELETE FROM orders` commits |
| 10:20 | 500 new orders created |
| 10:25 | Incident discovered |

Replicas may already have replayed the deletion, because replication faithfully copies valid committed operations, including mistakes.

PITR restores a base backup and replays archived WAL to a target before the destructive transaction. A recovery target immediately before the deletion can recover the original 3,000 orders.

But replacing production with that older snapshot would lose the 500 legitimate newer orders. A safer strategy is to restore into an isolated PostgreSQL instance, extract the deleted records, reconcile with current production, and import the missing records carefully.

PostgreSQL PITR does not normally allow us to replay arbitrary WAL transactions while skipping a selected committed DELETE. Selective business recovery requires reconciliation, not selective WAL skipping.

## 11. Business reconciliation after PITR

Suppose `ORD-101` was `PENDING` before deletion, but a separate Payment Service charged the customer afterward.

Restoring `ORD-101` as `PENDING` and blindly replaying a payment command could charge the customer twice.

A safer sequence is:

1. Restore missing order records into an isolated recovery environment.
1. Consult the Payment Service's authoritative payment state.
1. Reconcile the order status with durable payment events and business identifiers.
1. Apply idempotent state corrections without repeating external charges.
1. Verify related inventory, shipment, and payment records.
1. Resume affected workflows when their state is trustworthy.

Not every event log is a complete replayable source of truth. Event replay is safe only when the history and side-effect controls support it.

Recovery objectives for accidental corruption may differ from those for infrastructure failover. Restoring 2 TB and reconciling multiple services is not equivalent to promoting a ready standby within five minutes.

## 12. Base backups and WAL archiving

A base backup is a consistent physical copy of the PostgreSQL cluster. Archived WAL provides the changes required to recover from that backup to a later target.

```text
Base backup (Sunday)
      |
      +-- WAL segment 1
      +-- WAL segment 2
      +-- WAL segment 3
      +-- ... -> requested recovery point
```

Missing a required WAL segment can make the requested PITR target unreachable. Continuous WAL archiving does not automatically mean every recent acknowledged commit has already reached archive storage.

For a 2 TB order database, restoring from backup may take much longer than five minutes. A healthy standby is the primary mechanism for fast infrastructure failover. Backups are essential for historical recovery and cases where replication is insufficient.

### Backup placement and protection

Do not keep every copy in the same Availability Zone as the primary. Prefer independent failure domains, appropriate cross-region copies for regional disasters, encryption, restricted permissions, immutability or deletion protection, retention policies, and tested restores.

An unavailable zone does not necessarily mean its backups are permanently destroyed. But inaccessible backups cannot help meet an immediate recovery objective.

## 13. Disaster recovery drills

A design is not proven until we test it. A drill should simulate realistic failures and collect evidence.

For primary failover, measure failure detection, fencing, candidate selection, promotion, connection recovery, and application validation. Confirm every acknowledged order is present. For PITR, verify backup readability, WAL continuity, recovery target selection, restore duration, and business reconciliation.

A production drill checklist:

- [ ] Verify the failure scenario and intended RPO/RTO.
- [ ] Verify that the old writer is fenced.
- [ ] Verify standby durability and promotion eligibility.
- [ ] Measure time until correct application service is restored.
- [ ] Compare acknowledged order IDs against recovered data.
- [ ] Verify that retries do not create duplicate orders or charges.
- [ ] Test zone failure and coordination quorum loss.
- [ ] Restore backups into an isolated environment.
- [ ] Verify archived WAL continuity and retention.
- [ ] Record findings and retest failed recovery steps.

## 14. Common mistakes to avoid

| Mistake | Why it is dangerous |
| --- | --- |
| Promote the standby with the lowest CPU | It may lack acknowledged WAL |
| Assume etcd quorum means no data loss | Quorum coordinates authority, not PostgreSQL durability |
| Change the load balancer but do not fence | Old connections may still write |
| Trust a pre-freeze lease check | The lease can expire before the write executes |
| Treat a replica as a backup | Accidental deletes replicate too |
| Restore all production to an old PITR point | Legitimate newer transactions may disappear |
| Replay payment commands during recovery | Customers may be charged twice |
| Declare success because failover took 225 seconds | Missing acknowledged orders violate recovery correctness |

## 15. Revision questions

1. Why is the most advanced standby usually a better candidate than the healthiest but stale standby?
1. What can synchronous replication guarantee, and what does it not guarantee about every replica?
1. How does fencing prevent split-brain after a paused primary resumes?
1. Why does a three-node etcd cluster need two nodes to coordinate leadership?
1. What changes when two etcd nodes share a failed Availability Zone?
1. Why does a four-shard cluster not automatically allow another shard to take over missing shard data?
1. How are RPO and RTO measured differently?
1. Why does PITR to 10:14 not preserve orders created at 10:20?
1. Why must recovered order state be reconciled with the Payment Service?
1. Why are backup restore drills necessary even when synchronous replicas exist?

## Final mental model

- Replication keeps another copy of recent database changes available for failover.
- Quorum and fencing determine who is allowed to write.
- Backups and archived WAL let us reconstruct earlier database states.
- Business reconciliation makes recovered data correct across services.
- A recovery is successful only when both the required service and acknowledged-data guarantees are satisfied.

Next chapter: apply these principles to a hands-on PostgreSQL HA and disaster-recovery design, including measured failover tests and operational runbooks.
