# Chapter 2: Database Sharding and Consistent Hashing

## What you will learn

Sharding divides data across multiple independent databases. It can spread write traffic and storage requirements, but it also makes routing, querying, transactions, and migration harder.

## 1. Why shard an order database?

Suppose one PostgreSQL primary cannot safely handle our peak write load. We create five writable PostgreSQL shards. Each shard owns a subset of orders.

```text
                       Order Service
                             |
                      Shard Router
                /      /     |     \      \
               v      v      v      v      v
             DB 1   DB 2   DB 3   DB 4   DB 5
```

The router must answer: Where does `ORD-101` belong? Its answer must be consistent across application pods.

## 2. Choosing the shard key

A shard key is the value used to select a database. In our challenge, one enterprise customer produces 40% of all orders.

### Option A: `customerId`

If every order from customer `C-99` uses the same shard key, all of that customer's orders may land on one database. This creates a hot shard. The advantage is that retrieving all orders for one customer is easy because they are colocated.

### Option B: `orderId`

If each independent order uses its own ID, orders from the same large customer can spread across many shards. This is useful when writes need to scale and each order can be processed independently.

| Decision | Shard by customer ID | Shard by order ID |
| --- | --- | --- |
| Large customer's writes | May overload one shard | Can spread across shards |
| Query all orders for customer | Usually one shard | May require many shards |
| Per-order updates | Routed through customer key | Routed through order key |
| Customer-wide transactions | Easier to colocate | May cross shards |

For our scenario, `orderId` is the better starting point because write distribution is critical and orders are independent. This is a trade-off, not a universal rule.

## 3. What is a hot shard?

A hot shard receives much more work than the others. Imagine one shard gets 40% of 600 write TPS: it receives 240 TPS, above our tested 200 TPS maximum. Other shards might remain mostly idle.

Adding more shards does not solve skew unless the shard key and routing strategy spread the hot workload.

## 4. Modulo hashing and its problem

A simple router might calculate:

```java
int shardIndex = Math.floorMod(orderId.hashCode(), shardCount);
```

With five shards, the result ranges from 0 to 4. But when `shardCount` changes from five to six, many existing keys map to different shards. That means a large migration.

## 5. Consistent hashing

Consistent hashing places shard positions and key hashes on a logical ring. Each key belongs to the first eligible shard position encountered in a chosen direction around the ring.

```text
                   Shard A
                  /       \
             Key X         Shard B
             |                |
           Shard E         Key Y
              \              /
               Shard D -- Shard C
```

When a new shard position joins, it takes ownership of a portion of the ring from an existing owner. Most keys keep their original owner.

This reduces movement compared with simple modulo hashing, but does not copy data automatically.

## 6. Why virtual nodes matter

Physical shard positions are not guaranteed to be evenly spaced. A shard may own a much larger portion of the ring than another.

Virtual nodes give each physical shard multiple positions on the ring:

```text
Ring positions:
A1 -> C1 -> B1 -> A2 -> C2 -> B2 -> A3 -> ...

Physical owners:
A1, A2, A3 -> Shard A
B1, B2, B3 -> Shard B
C1, C2, C3 -> Shard C
```

Virtual nodes can improve balance and make redistribution more granular. They do not guarantee perfect balance. The number and placement of virtual nodes still matter.

## 7. Retrieving all orders for one customer

With `orderId` sharding, customer `C-99` might have orders on all five shards.

```text
GET /customers/C-99/orders
             |
        Query Service
        /  /  |  \  \
      DB1 DB2 DB3 DB4 DB5
        \  \  |  /  /
         Merge results
             |
       Sort + paginate
```

This is called a scatter-gather query. The query coordinator gathers results from multiple shards, combines them, sorts them, and returns the response.

Problems include:

- More network calls.
- Latency affected by the slowest shard.
- Harder global pagination and sorting.
- Higher load as the number of shards grows.

A possible optimization is a customer-to-order lookup index or dedicated read model. It adds complexity because the index must remain consistent with order changes.

## 8. Adding a sixth shard

Suppose `ORD-101` currently lives on Shard 2. After adding Shard 6, the new ring assigns it to Shard 6.

```text
Before: ORD-101 -> Shard 2 (record exists)
After:  ORD-101 -> Shard 6 (record not copied yet)
```

We cannot switch routing immediately. The record is still physically stored on Shard 2. We need a zero-downtime migration process that copies records, captures concurrent updates, and safely transfers ownership. Chapter 3 explains that process.

## 9. Production trade-offs

- Use a stable hash algorithm and an explicitly versioned ring configuration. Do not depend on language-specific hash behavior across services.
- Preserve routing history during migration so existing records remain findable.
- Plan for cross-shard queries and cross-shard transactions.
- Monitor per-shard TPS, storage, latency, and hot keys.
- Do not assume a perfectly even key distribution means an even workload. Some orders may be accessed more frequently than others.

## 10. Quick revision checklist

- [ ] Can I explain the difference between a shard and a read replica?
- [ ] Can I choose a shard key based on read and write patterns?
- [ ] Can I explain a hot shard?
- [ ] Can I explain why modulo hashing causes large remapping?
- [ ] Can I describe the purpose of virtual nodes?
- [ ] Can I explain why a routing change requires data migration?

## Practice question

One customer generates half of all writes. The system frequently queries individual orders by order ID but rarely needs the complete customer history. Would you prefer `customerId` or `orderId` as the shard key? Explain the write distribution and query trade-off.
