# Bloom Filter & HyperLogLog

A practical learning guide for software engineers and architects

## 01 / Bloom filter: membership

Imagine an entitlement server checking whether a device is registered. A database lookup for every request may be expensive. A Bloom filter can quickly rule out devices that are definitely absent, without storing every identifier.

### The core idea

A Bloom filter is a bit array plus k hash-derived positions. All bits start at zero. Inserting an element sets its positions to one. Checking an element reads those same positions.

```text
Index:   0 1 2 3 4 5 6 7 8 9
Initial: 0 0 0 0 0 0 0 0 0 0
Add A:   0 1 0 0 1 0 0 1 0 0   (1,4,7)
Add B:   0 1 1 0 1 0 0 1 0 1   (2,4,9)
```

| Query | Positions | Result                                |
| ----- | --------- | ------------------------------------- |
| A     | 1, 4, 7   | Possibly present (all ones)           |
| C     | 1, 5, 7   | Definitely absent (bit 5 is zero)     |
| D     | 2, 7, 9   | Possibly present, even if never added |

Device D demonstrates a false positive: different devices can set overlapping bits. A standard Bloom filter has no false negatives if all inserted elements are represented and the filter is intact and current.

### Remember this

One zero proves absence. All ones indicate possible presence, not proof. A Bloom filter is a fast gate before an authoritative lookup, not a replacement for that lookup.

## 02 / Bloom filter: engineering details

### Sizing and false positives

Let n be the expected number of elements, m the number of bits, k the number of hash positions and p the desired false-positive probability.

$$
\begin{aligned}m &= -\frac{n\ln p}{(\ln 2)^2}\\ k &\approx \frac{m}{n}\ln 2\\ p &\approx (1-e^{-kn/m})^k\end{aligned}
$$

For one million elements and a 1% false-positive target, allocate about 9.59 million bits (1.14 MiB) and use seven hash positions. If the actual population grows beyond the design capacity, the false-positive rate rises.

### Minimal Java-style pseudocode

```java
void add(String id) {
    for (int pos : positions(id)) bits.set(pos);
}
boolean mightContain(String id) {
    for (int pos : positions(id))
        if (!bits.get(pos)) return false;
    return true;
}
```

`positions()` must produce the same k well-distributed positions for a given ID. Add and query cost $O(k)$. Storage is $O(m)$ bits, fixed for a configured filter.

### Redis example (Bloom filter support required)

```redis
BF.RESERVE registered 0.01 1000000
BF.ADD registered device-A
BF.EXISTS registered device-A
```

A positive result must be confirmed in the database when correctness matters. A negative result can skip the database only when filter synchronization is guaranteed. Otherwise a newly registered device might be incorrectly rejected.

### Deletion and merging

Never clear a standard Bloom filter bit to delete an element: another element may depend on it. Counting Bloom filters support controlled deletion using counters. Compatible standard filters can be merged using bitwise OR; their capacity and false-positive rate must be reconsidered.

## 03 / HyperLogLog: the probability intuition

HyperLogLog (HLL) estimates cardinality: the number of distinct elements. Suppose 500 million API requests come from 50 million unique devices. A HashSet stores every distinct ID; HLL stores only statistical evidence about their number.

### The coin experiment

Toss a fair coin until the first tail. A run of 5 heads or more occurs about once in 32 experiments; 20 heads or more occurs about once in 1,048,576. Observing a rare pattern suggests many opportunities, but a lucky early result is possible.

| Leading zeros (z) | Probability of at least z | Rough scale |
| ----------------- | ------------------------- | ----------- |
| 1                 | 1/2                       | 2           |
| 5                 | 1/32                      | 32          |
| 10                | 1/1,024                   | 1,024       |
| 20                | 1/1,048,576               | 1,048,576   |

A well-distributed hash behaves like a sequence of coin tosses. The same device ID produces the same hash, so repeated requests do not create new statistical opportunities. A single maximum is too noisy; HLL uses many registers instead.

### Split the hash into two parts

```text
Hash:          01 | 000101
Bucket index:  01 = 1
Remaining:         000101
Leading zeros: 3
Rank:          3 + 1 = 4
```

With four buckets, the first two bits select a bucket. The remaining bits determine rank, defined as leading zeros plus one. The register stores `max(oldRank, newRank)`. The extra one distinguishes an observed rank from an empty register (zero).

## 04 / HyperLogLog: step-by-step registers

| Device  | Toy hash  | Bucket / rank | Registers 0-3 |
| ------- | --------- | ------------- | ------------- |
| Initial | -         | -             | 0, 0, 0, 0    |
| A       | 01 000101 | 1 / 4         | 0, 4, 0, 0    |
| B       | 10 100101 | 2 / 1         | 0, 4, 1, 0    |
| C       | 01 001011 | 1 / 3         | 0, 4, 1, 0    |
| D       | 00 001010 | 0 / 3         | 3, 4, 1, 0    |
| E       | 01 000001 | 1 / 6         | 3, 6, 1, 0    |
| A again | 01 000101 | 1 / 4         | 3, 6, 1, 0    |

Device C is distinct but does not change the register. HLL does not detect each unique element individually. It estimates the population from the overall register pattern.

### Java: educational register update

```java
int[] registers = new int[4];
void add(long hash) {
    int bucket = (int) (hash >>> 62);
    long remaining = hash << 2;
    int rank = Math.min(
        Long.numberOfLeadingZeros(remaining) + 1, 63);
    registers[bucket] = Math.max(registers[bucket], rank);
}
```

This assumes a high-quality 64-bit hash. The rank cap handles an all-zero suffix. This is only the update mechanism, not a production estimator.

### Why multiple buckets?

One rare hash can make a single maximum wildly misleading. Multiple buckets provide many noisy observations. Combining them reduces uncertainty. Increasing the number of registers fourfold approximately halves relative standard error.

## 05 / HyperLogLog: estimating the count

### Why the harmonic mean?

Consider estimates 10, 10, 10 and 1,000. The arithmetic mean is 257.5, but the harmonic mean is $4/(0.1+0.1+0.1+0.001)\approx 13.29$. It reduces the influence of unusually large estimates.

The classical raw HLL estimator is:

$$
E = \alpha_m\frac{m^2}{\sum_{i=1}^{m}2^{-M[i]}}
$$

Here m is the number of registers and alpha(m) is a bias correction constant. For an illustrative four-register sketch [3, 5, 5, 4], the reciprocal terms are 1/8, 1/32, 1/32 and 1/16. Their sum is 0.25. Using a toy alpha of 0.7 gives $0.7\times16/0.25=44.8$, or roughly 45 distinct elements.

The toy constant and four-register example are for intuition only. Production implementations use calibrated estimators and corrections, especially for small cardinalities.

### Memory versus accuracy

| Registers | 6-bit storage | Relative standard error |
| --------- | ------------- | ----------------------- |
| 1,024     | 768 bytes     | 3.25%                   |
| 4,096     | 3 KiB         | 1.63%                   |
| 16,384    | 12 KiB        | 0.81%                   |
| 65,536    | 48 KiB        | 0.41%                   |

Classical relative standard error is approximately $1.04/\sqrt{m}$. Redis uses 16,384 registers. At one million distinct devices, 0.81% corresponds to a typical error scale of about 8,125 devices, not a guaranteed error bound.

## 06 / Distributed analytics with Redis

### Merge registers, not counts

```text
Region A:  [3, 5, 2, 4]
Region B:  [4, 2, 6, 3]
Merged:    [4, 5, 6, 4]  (element-wise maximum)
```

Compatible HLLs must use the same hashing, seed, precision and register interpretation. Merging is idempotent, commutative and associative. Repeated or out-of-order snapshots converge to the same register maxima.

```redis
PFADD devices:region-a A B C
PFADD devices:region-b B C D
PFMERGE devices:global devices:region-a devices:region-b
PFCOUNT devices:global   # expected 4 in this example
```

### Architecture example

Three regions, 30 instances, 500 million requests/day, 50 million distinct devices/day, 90-day retention and a 1% relative standard error target.

```text
API instances -> Regional Redis (daily HLL)
              -> Snapshot aggregation service
              -> Global daily HLL -> Weekly/monthly union
```

Keep one daily sketch per region and one global daily sketch. Use UTC reporting boundaries and expire each sketch 90 days after its reporting day. Transfer compatible snapshots to a common Redis deployment before `PFMERGE`; it cannot directly merge independent deployments.

Register storage: $4\times90\times12\,\mathrm{KiB}\approx4.22\,\mathrm{MiB}$, excluding metadata and replicas. Average traffic is 5,787 requests/sec; use asynchronous updates or pipelining if synchronous analytics would add unacceptable latency. Peak traffic and loss tolerance need separate sizing.

If a region disconnects, mark global results incomplete. When it reconnects, merge the entire updated snapshot safely. Persistence and replication are still needed for durability.

## 07 / Choosing the right data structure

| Question            | HashSet      | Bloom filter        | HyperLogLog     |
| ------------------- | ------------ | ------------------- | --------------- |
| Who is present?     | Exact IDs    | Cannot list         | Cannot list     |
| Is A present?       | Exact        | Possibly / no       | Cannot answer   |
| How many distinct?  | Exact        | Not primary purpose | Estimate        |
| Individual deletion | Yes          | Not standard        | No              |
| Storage             | Grows with n | Sized for n and p   | Fixed precision |
| Merge               | Set union    | Bitwise OR*         | Register max*   |

Requires compatible configurations. Bloom filter union may increase false-positive probability.

### When to use which

Bloom filter: avoid unnecessary database or disk lookups, filter candidates, or reduce cache penetration. Never use a positive result alone to reject an order as a duplicate.

HyperLogLog: unique visitors, devices, searches and distributed usage analytics. Do not use it for billing, exact audits or idempotency enforcement.

HashSet or database uniqueness constraint: exact membership, exact counts or correctness-critical deduplication.

### Privacy and deletion

Neither standard sketch can remove a single historical element reliably. Filter excluded identifiers before ingestion for future observations. To remove historical contributions, rebuild affected sketches from permitted retained source data or discard the entire sketch. Hashing alone does not establish anonymity or regulatory compliance.

### Quick revision

Bloom filter: hash -> set bits -> check bits. One zero means definitely absent; all ones means possibly present.

HyperLogLog: hash -> split -> maximize rank -> combine -> estimate. It answers how many, not who.

### Further reading

- [Redis probabilistic data types](https://redis.io/docs/latest/develop/data-types/probabilistic/)
- Flajolet et al., HyperLogLog: the analysis of a near-optimal cardinality estimation algorithm (2007).
- Bloom, Space/time trade-offs in hash coding with allowable errors (1970).
