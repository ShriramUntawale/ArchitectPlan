# Chapter 1: Database Capacity Planning

## What you will learn

Capacity planning means estimating how much work a system must handle, measuring what its components can safely handle, and leaving enough spare capacity for spikes and failures. We will use the same e-commerce order system throughout.

## 1. Start with business traffic

The business expects 10,000 orders per minute on average and 18,000 orders per minute at peak. Convert both into orders per second:

$$
\text{Orders per second} = \frac{\text{Orders per minute}}{60}
$$

| Traffic | Orders per minute | Orders per second |
| --- | ---: | ---: |
| Average | 10,000 | 166.7 |
| Peak | 18,000 | 300 |

Why convert to seconds? Service throughput and database throughput are often measured per second, so this gives us a common unit.

## 2. Convert orders into database work

One order may cause several database transactions. In our exercise, assume two write transactions per order.

$$
\text{Peak write TPS} = 300 \times 2 = 600
$$

The database must handle 600 write transactions per second at peak. Do not confuse an HTTP request with a database transaction. One request might perform multiple SQL statements in one transaction, or several separate transactions.

### Example

A customer places order `ORD-101`:

1. The Order Service saves the order in PostgreSQL.
1. A later step reserves inventory in a database transaction.

For the exercise we count these as two write transactions. In a real design, the transactions might run on different services and different databases. Capacity must be calculated for each database separately.

## 3. Why 200 TPS does not mean 200 TPS is a safe operating target

Assume load testing shows that one PostgreSQL primary can sustain 200 transactions per second for this workload. Our target maximum utilization is 70%.

$$
\text{Usable TPS per primary} = 200 \times 0.70 = 140
$$

The remaining capacity is headroom. It gives the system space to absorb temporary traffic spikes, retries, maintenance, and changing query costs.

At the earlier average load of 167 transactions per second on a 200 TPS database:

$$
\text{Utilization} = \frac{167}{200} \times 100 = 83.5\%
$$

This leaves little room for growth. A queue may form when incoming work exceeds sustainable processing throughput. Near saturation, latency can rise sharply because requests compete for CPU, locks, I/O, and connections.

## 4. Estimate the number of writable shards

We need 600 write TPS at peak and want each primary to operate at no more than 140 TPS.

$$
\text{Required primaries} = \left\lceil \frac{600}{140} \right\rceil = 5
$$

With five evenly loaded writable primaries, each handles about 120 TPS, or 60% of its tested 200 TPS capacity.

| Shards | Average TPS per shard at peak | Utilization |
| ---: | ---: | ---: |
| 3 | 200 | 100% |
| 4 | 150 | 75% |
| 5 | 120 | 60% |
| 6 | 100 | 50% |

Five is a mathematical minimum under our assumptions, not a final production recommendation. Real traffic may be uneven, and a shard failure can redistribute work.

## 5. Why read replicas do not solve write saturation

A PostgreSQL read replica can serve suitable read queries, but it does not provide another independently writable primary for the same data. If our primary is saturated by inserts and updates, adding read replicas alone will not distribute those writes.

```text
             Order Service
               /       \
        Read requests   Write requests
             |                |
       Read replicas    Primary database
```

Read replicas also introduce possible replication lag. A customer who has just updated an order may need to read from the primary to see the latest committed state.

## 6. How to validate capacity in production

Do not decide based only on theoretical TPS. Run load tests with realistic SQL queries, indexes, transaction sizes, concurrency, and data volume.

Measure:

- Transaction throughput and error rate.
- p95 and p99 transaction latency.
- CPU, memory, disk IOPS, and WAL throughput.
- Database connection pool utilization.
- Lock waits, deadlocks, and slow queries.
- Queue depth, retries, and downstream timeouts.

Increase traffic gradually. A system that handles 600 TPS for ten seconds might not sustain it for an hour.

## 7. Backpressure versus scaling

Backpressure protects an overloaded component by limiting or delaying incoming work. It does not create extra database throughput.

For example, if Kafka receives 167 events per second but consumers can process only 120, the backlog grows by about 47 events per second. Slowing consumption can protect PostgreSQL, but the backlog grows until we increase sustainable capacity or reduce incoming work.

## 8. Production trade-offs

- Scale up the primary when better hardware or query optimization provides enough headroom.
- Use read replicas when reads dominate and eventual consistency is acceptable.
- Consider sharding when writes or data volume exceed what a single primary can reasonably handle.
- Account for shard skew, failover headroom, and operational overhead before choosing a shard count.

## 9. Quick revision checklist

- [ ] Can I convert orders per minute into orders per second?
- [ ] Can I distinguish API requests, SQL statements, and database transactions?
- [ ] Can I calculate utilization and headroom?
- [ ] Can I explain why read replicas do not distribute primary writes?
- [ ] Can I describe the load test needed before approving scaling?

## Practice question

Peak traffic increases to 24,000 orders per minute. Each order still requires two write transactions. Each primary supports 200 TPS and the target is 70% utilization. How many evenly loaded writable primaries do you need? Calculate it before reading the next chapter.

Answer for self-check: 24,000 / 60 = 400 orders/sec, 800 write TPS, and ceiling(800 / 140) = 6 primaries.
