# Kafka Architecture Challenges 1-5: A Practical Learning Guide

This guide covers the first five architecture challenges from our learning sessions. It focuses on Kafka reliability, event ordering, transactional outbox, concurrent publishers, partitioning, and consumer scaling. Database capacity planning and shard migration are intentionally reserved for a separate guide.

## The big picture

Imagine an online store. The Order API accepts an order, Kafka carries events, and services update PostgreSQL. The main goal is to process every accepted order without losing it or applying the same business action twice.

```text
Customer -> Order API -> Kafka -> Order Consumer -> PostgreSQL
                                   |
                                   +-> Inventory / downstream services
```

Kafka normally provides at-least-once processing when a consumer commits offsets after successful processing. This means an event may be delivered again. Your application must be prepared for duplicates.

## Challenge 1: Consumer crashes before committing its Kafka offset

### The situation

1. Kafka delivers `OrderCreated(orderId=ORD-101)`.
1. The consumer inserts `ORD-101` into PostgreSQL.
1. The consumer crashes before committing its Kafka offset.
1. Kafka redelivers the same event after the consumer restarts.

### What can go wrong?

If the consumer inserts the order again, it may create duplicates. If it treats every database error as success, it might skip work that never completed.

### The solution: Idempotent business processing

Give each order a stable business ID and enforce a unique constraint in PostgreSQL. For an order and its inventory change stored in the same database, apply both changes in one transaction. An inbox or processed-event record can identify completed events.

```sql
BEGIN;
INSERT INTO processed_events(event_id) VALUES (:event_id)
ON CONFLICT DO NOTHING;
-- Only when the event was newly claimed:
-- insert order and reserve inventory in this same transaction.
COMMIT;
```

The example is a sketch: the application must check whether the insert created a row, and only then execute the business changes. Do not catch a duplicate-key error and blindly run every downstream step again.

Commit the Kafka offset only after the database transaction succeeds, or after verifying that the event was already fully processed. If the database commit succeeds but offset commit fails, replay is safe because the inbox and business changes were committed together.

### Key lesson

A unique order ID prevents duplicate orders. Transactional event processing prevents duplicate business effects. These are related but different guarantees.

## Challenge 2: Order saved but Kafka event never published

### The situation

The Order Service saves an order and then publishes `OrderCreated`. PostgreSQL commits, but the service crashes before sending the event. Inventory never learns about the order.

### The solution: Transactional outbox

Write the order and an outbox event in the same PostgreSQL transaction.

```text
Order API
  -> PostgreSQL transaction
       -> INSERT orders
       -> INSERT outbox(event_id, payload, status=PENDING)
  -> COMMIT

Outbox publisher -> Kafka -> Inventory consumer
```

A background publisher reads pending outbox records, sends them to Kafka, waits for acknowledgement, and then marks them as published.

If Kafka accepts the event but the publisher crashes before marking it published, the publisher may send it again. That is acceptable only if the downstream consumer handles duplicates safely.

### Inbox pattern on the consumer

The Inventory Service stores a processed event ID and applies the inventory update in the same local database transaction. A replayed event cannot deduct stock twice. Kafka offset acknowledgement follows the successful database commit.

### Key lesson

The outbox prevents a committed database change from losing its associated event. The inbox prevents duplicate deliveries from repeating business effects. Outbox publication is normally at least once, not exactly once.

## Challenge 3: Multiple Kubernetes pods publish from one outbox

### The situation

Five Order Service pods each execute:

```sql
SELECT * FROM outbox WHERE status = 'PENDING' LIMIT 100;
```

All five can select the same events. Dividing work into batches does not by itself prevent overlap.

### Step 1: Atomically claim different rows

PostgreSQL supports `FOR UPDATE SKIP LOCKED`. It lets workers skip rows locked by other transactions.

```sql
WITH candidates AS (
    SELECT event_id FROM outbox
    WHERE status = 'PENDING'
       OR (status = 'PROCESSING' AND lease_until < now())
    ORDER BY created_at, event_id
    LIMIT 100
    FOR UPDATE SKIP LOCKED
)
UPDATE outbox AS o
SET status = 'PROCESSING',
    claimed_by = :worker_id,
    claim_version = o.claim_version + 1,
    lease_until = now() + interval '30 seconds'
FROM candidates AS c
WHERE o.event_id = c.event_id
RETURNING o.*;
```

Run this as a short transaction and commit before calling Kafka. The row locks prevent overlapping claims during that transaction. The durable lease controls ownership afterward.

### Step 2: Recover abandoned work

Suppose Pod A claims event E103 for 30 seconds and crashes. Once its lease expires, Pod B can reclaim E103. The timeout should reflect real processing latency, retries, and pauses; long-running work may need lease renewal.

### Step 3: Fence stale workers

Suppose Pod A pauses, its lease expires, and Pod B reclaims E103. Pod A later resumes. Both might try to mark the event published.

Store a claim generation (`claim_version`). Every new claim increments it. The final update must verify the worker and version:

```sql
UPDATE outbox
SET status = 'PUBLISHED', published_at = now(),
    claimed_by = NULL, lease_until = NULL
WHERE event_id = :event_id
  AND status = 'PROCESSING'
  AND claimed_by = :worker_id
  AND claim_version = :claim_version
  AND lease_until > now();
```

If zero rows are updated, the worker no longer owns the event. This prevents stale ownership updates in PostgreSQL, but it cannot undo a Kafka event already published by a stale worker. The consumer must remain idempotent.

### Key lesson

Use atomic claims to distribute work, leases to recover crashes, fencing versions to reject stale workers, and idempotency to handle duplicate publication.

## Challenge 4: A hot Kafka partition and event ordering

### The situation

There are 12 Kafka partitions and 12 consumers. Messages use `customerId` as the Kafka key. One customer generates 60% of the orders. That customer's messages hash to the same partition, creating a hot partition.

Adding consumers alone cannot split a single partition among consumers in the same consumer group.

### Choose the key based on the ordering requirement

If different orders are independent, use `orderId` as the key. For example:

```text
ORD-101 -> Partition 2: Created, Paid, Reserved, Confirmed
ORD-102 -> Partition 7: Created, Paid, Reserved, Confirmed
ORD-103 -> Partition 9: Created, Paid, Reserved, Confirmed
```

Different orders can be processed in parallel, while events for the same order land on the same partition, assuming the partition count and partitioning scheme remain stable.

Kafka preserves record order within a partition. It does not guarantee that asynchronous application tasks finish in the same order. If the consumer launches multiple threads, it must preserve per-order sequencing where required.

Also, increasing partition count can remap keys and complicate ordering during a migration. Plan partition changes carefully.

### Key lesson

Do not select a partition key merely because it is convenient. First ask: What must be ordered together? Per-order ordering often provides better distribution than per-customer ordering.

## Challenge 5: Consumer lag, backpressure, and scaling

### The situation

- Incoming traffic: 10,000 events/minute = about 166.7 events/second.
- Kafka partitions: 12.
- Consumer pods: 12.
- Capacity per consumer: 10 events/second.
- Total processing capacity: 12 x 10 = 120 events/second.

Backlog grows by about 46.7 events/second, or 2,800 events/minute. At constant rates, that is about 28,000 queued events after 10 minutes.

### Why 24 pods alone will not fix it

With 12 partitions and 24 consumers in one consumer group, only up to 12 consumers can be assigned partitions at once. The remaining consumers do not increase partition-level processing throughput.

### What can improve throughput?

- Increase partition count when key distribution and ordering allow it.
- Scale active consumers with available partitions.
- Optimize slow processing, database queries, and external calls.
- Measure downstream database capacity before increasing concurrent writes.
- Use batching where safe and beneficial.

If there are 24 partitions and 24 consumers, theoretical capacity at 10 events/second each is 240 events/second. That exceeds incoming traffic, but only if the database and other dependencies can keep up.

### What backpressure does and does not do

Backpressure limits the rate of work admitted to a stressed component. It protects downstream services from overload, but it does not create capacity. Slowing consumption while input stays high generally increases Kafka lag.

Monitor consumer lag, oldest-event age, throughput, processing latency, retries, database utilization, and downstream saturation. Test scaling changes rather than assuming linear gains.

### Key lesson

Capacity is constrained by the slowest required component. More consumers help only when partitions, key distribution, and downstream systems permit greater throughput.

## How the five challenges connect

| Failure or requirement | Main technique | What it does not solve alone |
| --- | --- | --- |
| Kafka redelivers an event | Idempotent processing and unique IDs | Missing downstream work |
| DB commits but event is not sent | Transactional outbox | Duplicate publication |
| Multiple publishers claim same event | `SKIP LOCKED` and short claim transactions | Crash recovery after claim |
| Claimed worker crashes | Lease expiration | Stale worker actions |
| Old worker resumes | Claim version / fencing | Already-sent Kafka duplicates |
| One Kafka partition is overloaded | Better partition key | Database bottlenecks |
| Consumer lag grows | Capacity analysis and scaling | Overloaded downstream systems |
| Downstream service is overloaded | Backpressure | Insufficient long-term capacity |

## Practical review checklist

- [ ] Every event has a stable unique ID.
- [ ] Kafka offsets are acknowledged only after safe processing.
- [ ] Related local database changes are atomic.
- [ ] Order and outbox inserts share one database transaction.
- [ ] Consumers handle duplicate delivery without duplicate business effects.
- [ ] Concurrent publishers claim disjoint work atomically.
- [ ] Expired claims can be recovered.
- [ ] Stale workers cannot overwrite current claim state.
- [ ] Kafka partition keys match business ordering needs.
- [ ] Consumer parallelism does not break required event order.
- [ ] Consumer capacity exceeds expected sustained arrival rate with headroom.
- [ ] Downstream systems are tested before scaling consumer concurrency.
- [ ] Lag, retries, oldest event age, and database saturation are monitored.

## Final mental model

Think of Kafka as a durable conveyor belt. Offsets track how far a consumer has progressed. The outbox ensures a committed business change gets onto the belt. The inbox ensures that seeing the same package twice does not repeat the business action. Partition keys determine which lane a package enters and which other packages it must stay ordered with. Leases and fencing help multiple publisher workers cooperate safely. Capacity planning ensures packages do not arrive faster than the whole system can process them.

Next document (separate): database capacity planning, PostgreSQL write sharding, consistent hashing, and zero-downtime shard migration.
