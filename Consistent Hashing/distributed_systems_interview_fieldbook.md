# Distributed Systems Interview Fieldbook

A practical, easy-to-revise guide based on our discussion of NGINX, Spring Boot, PostgreSQL, Redis, Kafka, idempotency, and Sagas.

Read the cheat sheet in 5 minutes, the architecture in 15 minutes, or the complete guide in 45-60 minutes.

## Contents

1. [Five-minute cheat sheet](#page-1---five-minute-cheat-sheet)
2. [The big picture](#page-2---the-big-picture)
3. [Consistent hashing and NGINX](#page-3---consistent-hashing-and-nginx)
4. [Scaling PostgreSQL](#page-4---scaling-postgresql)
5. [Read-after-write consistency](#page-5---read-after-write-consistency)
6. [Idempotency and concurrent requests](#page-6---idempotency-and-concurrent-requests)
7. [Kafka, inbox, and transactional outbox](#page-7---kafka-inbox-and-transactional-outbox)
8. [Saga versus two-phase commit](#page-8---saga-versus-two-phase-commit)
9. [Saga state machines and race conditions](#page-9---saga-state-machines-and-race-conditions)
10. [End-to-end bank transfer design](#page-10---end-to-end-bank-transfer-design)
11. [Failure drills and interview answers](#page-11---failure-drills-and-interview-answers)
12. [Decision matrix and final recap](#page-12---decision-matrix-and-final-recap)

---

# Page 1 - Five-minute cheat sheet

| Concept | Explain it in one sentence | Common trap |
| --- | --- | --- |
| Consistent hashing | Assign keys to nodes with minimal remapping when nodes change. | Does not copy local state to another server. |
| Virtual nodes | Give each server multiple positions on the hash ring to improve balance. | They do not eliminate movement or hot keys. |
| NGINX affinity | Hash a stable request key to choose an upstream. | Client-controlled headers are not trustworthy identity by default. |
| Read replicas | Move suitable reads away from the primary database. | Replicas can be stale. |
| Read-after-write | A user must see their own committed update. | A fixed Redis TTL is not a strict guarantee. |
| PostgreSQL WAL/LSN | Track the progress of database replication. | Low time lag does not prove a specific write was replayed. |
| Idempotency | Repeated requests produce one committed business effect. | Check-then-insert has a race condition. |
| Unique constraint | Let the database arbitrate duplicate keys. | It does not enforce a valid business state transition. |
| Transactional outbox | Commit business changes and an event record together. | Publishing may happen more than once. |
| Inbox/deduplication | Record a consumed operation in the same transaction as its effect. | Event ID alone may not identify the logical business operation. |
| Kafka offset | Records a consumer's progress. | Acknowledging before database commit can lose processing. |
| Saga | Coordinate multiple local transactions with retries and compensation. | Intermediate states are possible. |
| 2PC | Coordinate atomic commit across multiple transactional resources. | Blocking, locking, and operational cost. |
| Conditional update | Change state only when its current state permits it. | Must be in the same transaction as related business effects. |
| Reconciliation | Compare durable records to resolve uncertain outcomes. | A status read alone can race with a later write. |

Interview answer template: requirement -> failure/bottleneck -> mechanism -> correctness guarantee -> trade-off.

---

# Page 2 - The big picture

## How our discussion evolved

We began with a Spring Boot application handling many users. That question expanded into multiple independent scaling and correctness problems.

```text
Clients
   |
   v
NGINX -- consistent hashing / load balancing
   |
   +------ Spring Boot A
   +------ Spring Boot B
   +------ Spring Boot C
                 |
           Database access
                 |
       +---------+----------+
       |                    |
  PostgreSQL primary    Read replicas
       |
       +-- Outbox -> Kafka -> Other services
                           |
                    Saga coordinator
                           |
                    Durable transfer state
```

## Separate the responsibilities

- NGINX answers: which application instance should receive this request?
- Spring Boot answers: which business operation should run?
- PostgreSQL answers: what state has committed durably?
- Redis answers: what short-lived shared information can be accessed quickly?
- Kafka answers: how can events be delivered asynchronously?
- Saga coordination answers: what happens when a multi-step business operation partially succeeds?

Key idea: distributing traffic does not automatically distribute database data, replicate state, or guarantee financial correctness.

---

# Page 3 - Consistent hashing and NGINX

## Use case

Requests for the same device benefit from landing on the same Spring Boot instance because that instance maintains a warm local cache.

## Why modulo hashing hurts

A naive mapping uses $hash(key) \bmod N$. When $N$ changes, most key-to-server mappings may change. With consistent hashing, the hash space stays stable, so adding one equally weighted server to 100 ideally remaps about $1/101$, or 0.99%, of keys.

```text
Key -> hash -> position on ring -> next server clockwise
```

Virtual nodes give each physical server multiple ring positions. They improve distribution, not the fundamental minimal-remapping property.

## NGINX example

```nginx
upstream entitlement_backend {
    hash $http_x_device_id consistent;
    server app1:8080 max_fails=3 fail_timeout=30s;
    server app2:8080 max_fails=3 fail_timeout=30s;
    server app3:8080 max_fails=3 fail_timeout=30s;
}
server {
    listen 80;
    location / {
        proxy_pass http://entitlement_backend;
        proxy_set_header X-Device-ID $http_x_device_id;
    }
}
```

This is an illustrative configuration. In production, authenticate the device identity at a trusted boundary, define missing-key behavior, configure timeouts and retries, and test failure handling. NGINX consistent hashing uses an internal continuum; you do not manually configure virtual nodes.

## Caveats

- Affinity is not a session store. If App2 crashes, its in-memory cache disappears.
- `$request_uri` groups by URL, `$remote_addr` by observed IP, and `$http_x_device_id` by device header.
- NAT can make many users share one IP.
- An attacker may forge a device header unless it is validated or set by a trusted gateway.
- Hot keys can overload one instance even if hash ranges are balanced.
- If all instances are stateless, round-robin may be easier to operate.

Interview answer: "I use consistent hashing when stable key ownership improves cache locality. It reduces remapping during scale changes, but it is not state replication or a database scaling strategy."

---

# Page 4 - Scaling PostgreSQL

## Use case

Five Spring Boot instances serve two million users, but most database traffic is reads.

## Scaling sequence

1. Measure slow queries, indexes, connection pools, and database saturation.
2. Add caching for frequently accessed, safe-to-cache data.
3. Add read replicas if read traffic is the bottleneck.
4. Consider sharding only when a single primary or dataset can no longer meet requirements.

```text
Spring Boot -> read/write router
                  |
             +----+-----+
             |          |
          Primary    Read replicas
          writes     eligible reads
```

## Sharding versus replication

- Replication creates copies of data. It helps read capacity and recovery but not primary write throughput by itself.
- Sharding divides data ownership across databases. It may increase write capacity but makes joins, transactions, migrations, and rebalancing harder.
- A consistent-hash ring can choose a shard, but changing the ring does not physically move existing records. Resharding requires controlled data migration and cutover.

## Spring Boot routing caveat

`@Transactional(readOnly = true)` can guide routing but does not guarantee replica freshness. With `AbstractRoutingDataSource`, route before acquiring a physical connection. A lazy connection proxy can help with transaction-aware selection.

Interview answer: "For a read-heavy service, I first optimize queries and add replicas, while routing consistency-sensitive reads to the primary. I do not shard just because the user count is large."

---

# Page 5 - Read-after-write consistency

## Use case

A user updates a profile photo and immediately refreshes the page. The replica may still hold the old value.

```text
T0: primary commits new photo
T1: replica has old photo
T2: user reads replica -> stale result
T3: replica catches up
```

## Options

| Option | Benefit | Caveat |
| --- | --- | --- |
| Read primary after a write | Straightforward freshness | More primary load |
| Redis recent-write marker | Shares affinity information across app instances | Fixed TTL is a heuristic, not a guarantee |
| Version/freshness token | Explicitly expresses the minimum required state | Requires correct propagation and verification |
| WAL LSN fence | Select replicas that replayed at least the required position | Must obtain a safe post-commit position and handle failover |

A PostgreSQL replica can expose replay progress with:

```sql
SELECT pg_last_wal_replay_lsn();
```

A WAL position is not a wall-clock lag. If a read requires a write covered by LSN `0/500`, a replica at `0/490` is not eligible. Among eligible replicas, choose by load or latency. If none qualify, wait within a deadline or use the primary.

## Redis crash gap

If App1 commits PostgreSQL but crashes before writing a Redis marker, App3 may not know that a recent write occurred. This is a dual-write failure. Strict freshness needs a durable, correctly propagated requirement or a primary read. An asynchronous outbox alone does not make an immediate read fresh.

Interview answer: "First filter replicas by correctness, then load balance among the eligible ones."

---

# Page 6 - Idempotency and concurrent requests

## Use case

A transfer commits, but the HTTP response is lost. The client retries.

```http
POST /transfers
Idempotency-Key: ABC123
Content-Type: application/json

{"from":101,"to":202,"amount":10000}
```

The same key with the same request must not transfer the money again. A different key normally represents a new operation. The same key with different parameters should be rejected as a conflict.

## Why a pre-check is unsafe

```text
App1: key absent -> transfer
App2: key absent -> transfer
```

Both checks can succeed before either insert commits. Use a database unique constraint and transactional coordination, not only `existsByKey()`.

```sql
CREATE TABLE transfer_requests (
    idempotency_key TEXT PRIMARY KEY,
    request_hash TEXT NOT NULL,
    status TEXT NOT NULL,
    result_json JSONB
);
```

A complete implementation atomically claims the key, verifies the request hash, performs the transfer, and stores the result. A competing request either waits or receives an explicit `IN_PROGRESS` response. It must never receive premature success.

## Rollback behavior

- Committed: return stored result on retry.
- Rolled back: allow a safe retry of the logical operation.
- Unknown/in progress: resolve outcome before retrying the money movement.

Do not infer transfer success from a balance check. Other transfers and stale replicas make that ambiguous.

---

# Page 7 - Kafka, inbox, and transactional outbox

## The dual-write problem

```text
Debit DB COMMIT succeeds
       |
App crashes before Kafka publish
       |
Coordinator never hears about debit
```

## Transactional outbox

Write the business effect and the event record in one local database transaction.

```sql
BEGIN;
-- Validate funds and apply debit atomically.
-- Persist unique debit operation for this transfer.
INSERT INTO outbox_events(event_id, transfer_id, event_type)
VALUES ('E1', 'T123', 'DEBIT_COMPLETED');
COMMIT;
```

An outbox worker or CDC publisher later sends the event to Kafka. Publication is typically at least once, so duplicates are possible.

## Idempotent inbox

The Credit Service must atomically claim the logical credit operation and update the account in one PostgreSQL transaction.

```sql
BEGIN;
INSERT INTO processed_operations(operation_id)
VALUES ('T123:CREDIT')
ON CONFLICT DO NOTHING;
-- Only if a row was inserted: credit account and write CREDIT_COMPLETED outbox event.
COMMIT;
```

A unique logical operation ID is stronger than relying only on a delivery event ID, since different event IDs might refer to the same requested credit.

## Kafka acknowledgment order

```text
Consume event
   -> commit business DB transaction
   -> acknowledge Kafka offset
```

If the process crashes after DB commit but before offset acknowledgment, Kafka can redeliver. The inbox prevents repeating the business effect. If the offset is committed before the DB transaction, processing can be lost.

Interview answer: "Kafka may deliver more than once. I guarantee one committed database effect through atomic deduplication, then acknowledge the message."

---

# Page 8 - Saga versus two-phase commit

## Use case

Debit Account A in PostgreSQL A, then credit Account B in PostgreSQL B. A normal local `@Transactional` does not make both databases commit atomically.

## Saga

```text
INITIATED -> DEBITED -> CREDIT_PENDING -> COMPLETED
                                |
                                +-> CANCELLING -> COMPENSATED
```

Each step is a local transaction. Events coordinate steps. On failure, retry or perform a compensating business action. Partial committed states can exist.

## Two-phase commit

```text
Coordinator -> PREPARE both databases
            -> COMMIT both, or decide rollback
```

2PC coordinates atomic commit across participants but may hold resources and block during failures. It does not automatically give arbitrary external readers a synchronized cross-database snapshot.

| Requirement | Better starting point |
| --- | --- |
| Atomic outcome across participating databases | 2PC or redesign around one transactional boundary |
| Independent services with acceptable pending states | Saga |
| Fast, highly available asynchronous workflow | Saga with durable recovery |
| Never expose a partial business outcome | Define visibility rules, ledger semantics, and atomicity requirements before choosing |

For money movement, a durable ledger, reconciliation, reservations or pending funds, and explicit business invariants are essential regardless of orchestration style.

---

# Page 9 - Saga state machines and race conditions

## The late-event problem

```text
Debit A -> credit B times out -> refund A
                             -> delayed credit B executes
```

Now both accounts may have received money. A DLQ replay is not permission to execute an obsolete credit.

## Cancellation handshake

1. Coordinator enters `CANCELLING` but does not refund immediately.
2. Credit Service atomically resolves the operation as `CREDITED` or `CANCELLED`.
3. If `CREDITED`, coordinator reconciles and must not refund.
4. If `CANCELLED`, future credit attempts are blocked and compensation can proceed.
5. If uncertain, remain in a recovery state and reconcile.

## Atomic conditional transitions

```sql
UPDATE credit_operations
SET status = 'CREDITED'
WHERE transfer_id = 'T123' AND status = 'PENDING';
```

```sql
UPDATE credit_operations
SET status = 'CANCELLED'
WHERE transfer_id = 'T123' AND status = 'PENDING';
```

Only one can successfully change the same `PENDING` row, under normal PostgreSQL transactional locking. Check affected-row counts. Put the winning transition and corresponding money movement or outbox update in the same local transaction.

A unique transfer ID prevents duplicate rows. It does not by itself prevent competing updates to an existing row.

## Out-of-order events

If `CREDIT_COMPLETED` is already durably confirmed, a later `CREDIT_FAILED` must not trigger a refund. If `CREDIT_FAILED` arrives first, do not assume credit never happened. Reconcile the Credit Service's durable state and use the cancellation handshake. A plain status query can race with a subsequent credit.

---

# Page 10 - End-to-end bank transfer design

## Happy path

```text
Client POST /transfers (idempotency key)
  |
Saga coordinator stores INITIATED
  |
Debit service: debit/reserve + operation record + outbox (one commit)
  |
Outbox publisher -> Kafka DEBIT_COMPLETED
  |
Coordinator records CREDIT_PENDING and issues credit command
  |
Credit service: dedup + credit + outbox (one commit)
  |
Outbox publisher -> Kafka CREDIT_COMPLETED
  |
Coordinator: conditional COMPLETE + outbox (one commit)
  |
Kafka TRANSFER_COMPLETED -> notification / status consumers
```

## Data ownership

| Component | Durable responsibility |
| --- | --- |
| Coordinator DB | Transfer lifecycle and decisions |
| Debit DB | Debit/hold operation, account state, outbox |
| Credit DB | Credit/cancel decision, account state, inbox, outbox |
| Kafka | Event transport and replay within retention |
| Redis | Optional cache/coordination, not authoritative transfer ledger |

## Financial invariants

- A logical debit and credit must each have at most one committed effect.
- A transfer cannot be both durably credited and durably cancelled.
- `COMPLETED` requires confirmed debit and credit.
- A refund must not occur while a late credit can still execute.
- All uncertain transfers must be discoverable by reconciliation.
- Double-entry ledger entries should balance and remain auditable.

---

# Page 11 - Failure drills and interview answers

| Failure | Correct recovery |
| --- | --- |
| NGINX backend dies | Route to another healthy backend; rebuild lost local cache from durable state. |
| Replica is behind | Use primary or a replica proven to satisfy the freshness requirement. |
| App crashes after DB write before Redis marker | Do not treat Redis TTL as a strict consistency guarantee. |
| HTTP transfer response lost | Retry with same idempotency key; return recorded outcome. |
| Two requests use same key simultaneously | Database unique claim and transactional arbitration. |
| Debit commits before Kafka publish | Transactional outbox republishes committed event. |
| Kafka redelivers credit event | Transactional inbox/dedup prevents second credit. |
| Credit commits before Kafka acknowledgment | Redelivery is safe; stored credit effect is not repeated. |
| Credit commits before CREDIT_COMPLETED publish | Credit outbox eventually publishes confirmation. |
| Coordinator crashes before recording completion | Replay or reconcile, then conditionally complete. |
| Credit DB down for 30 minutes | Retry with policy; coordinate cancellation before compensation. |
| Late credit after refund | Must be blocked by durable credit-versus-cancel arbitration. |
| CREDIT_FAILED arrives after CREDIT_COMPLETED | Do not reverse a confirmed credit; reconcile conflict. |
| State changed but balance update fails in same transaction | Both changes roll back on a rollback-triggering failure. |

## Interview questions with short answers

### Why not use consistent hashing everywhere?

Because it solves stable mapping, not freshness, persistence, atomicity, or balanced traffic for hot keys.

### Why aren't read replicas enough for strong consistency?

Asynchronous replication can lag. Route sensitive reads to primary or enforce a replay/version fence.

### Why not check the balance before retrying a transfer?

Balances can change for unrelated reasons and replicas can be stale. Query the specific transfer by durable idempotency/operation ID.

### What does an outbox guarantee?

A committed business change has a durable event record available for eventual publication. It does not guarantee exactly-once Kafka delivery.

### What is the difference between inbox and outbox?

Inbox protects consumers from repeated business effects. Outbox protects producers from losing committed events.

### Why is a Saga not a distributed ACID transaction?

Each local step commits separately. Compensation is another business transaction, not an automatic rollback of all shards.

### Why isn't `status = CANCELLING` enough to refund?

The credit may already have committed or may still execute. Cancellation must be durably confirmed by the credit owner before refund.

---

# Page 12 - Decision matrix and final recap

| If you see... | Consider... | Verify... |
| --- | --- | --- |
| Scale-up remaps cached keys | Consistent hashing | Stable trusted key, hotspots, failover |
| Primary overloaded by reads | Read replicas/cache | Query metrics, freshness requirements |
| User sees old data after update | Read-after-write routing | Replica position or primary fallback |
| Duplicate HTTP requests | Idempotency key | Unique DB claim and request hash |
| DB committed but event missing | Transactional outbox | Atomic insert and reliable publisher |
| Duplicate Kafka messages | Inbox/dedup | One transaction with business effect |
| Multi-database workflow | Saga or 2PC | Partial visibility and atomicity requirement |
| Delayed or contradictory events | State machine and reconciliation | Durable state, conditional transitions |
| Compensation races with credit | Cancellation handshake | Atomic credit-or-cancel decision |

## One-minute final explanation

"I separate scalability from correctness. NGINX consistent hashing improves routing affinity, while PostgreSQL replicas handle suitable reads. Read-after-write paths need a freshness guarantee, not only a low-lag estimate. I use durable idempotency keys for retried commands, transactional outbox for reliable event publication, and inbox deduplication for Kafka consumers. For cross-database workflows, a Saga coordinates local transactions with explicit pending, completion, cancellation, and compensation states. Conditional database updates and reconciliation prevent conflicting late events from causing double credit or unsafe refunds. I choose 2PC instead when coordinated atomic commit is a hard requirement and its operational trade-offs are acceptable."

## Final mental model

```text
ROUTING       -> Consistent hashing
READ SCALE    -> Replicas and caching
FRESHNESS     -> Primary or verified replica
DUPLICATES    -> Idempotency and uniqueness
EVENT LOSS    -> Transactional outbox
REDELIVERY    -> Inbox and offset discipline
MULTI-DB FLOW -> Saga or 2PC
RACES         -> Atomic state transitions
UNCERTAINTY   -> Reconciliation
```

End of fieldbook.
