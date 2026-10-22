---
tags: [system-design, sharding, databases, partitioning, faang]
aliases: [Sharding, Horizontal Partitioning, Data Partitioning]
created: 2026-10-22
difficulty: ⭐⭐⭐
type: concept
---

# 13 — Database Sharding

> [!important] Core Idea
> Sharding (horizontal partitioning) splits data across multiple database nodes. Each shard holds a subset of data. This enables write scaling beyond what a single node can handle — but adds significant complexity.

---

## Why Shard?

```
Single DB node limits:
  → Storage: Can't store > few TB on one node efficiently
  → Write throughput: Limited by single node CPU/disk
  → Memory: Can't cache entire dataset

Sharding solves:
  → Each shard stores a portion of data
  → Writes distributed across shards
  → Each shard's working set fits in memory
```

**Before sharding, try:**
1. Read replicas (for read-heavy)
2. Caching (reduce DB load)
3. Vertical scaling
4. Query optimization
5. Archiving old data

---

## Sharding Strategies

### 1. Hash-Based Sharding
```
shard_id = hash(shard_key) % number_of_shards

Example: user_id → hash(user_id) % 4
  user 123 → hash(123) = 789 → 789 % 4 = 1 → Shard 1
  user 456 → hash(456) = 234 → 234 % 4 = 2 → Shard 2
```

```mermaid
graph LR
    Client --> Router[Shard Router]
    Router -->|hash(user_id) % 4 = 0| S0[(Shard 0)]
    Router -->|hash(user_id) % 4 = 1| S1[(Shard 1)]
    Router -->|hash(user_id) % 4 = 2| S2[(Shard 2)]
    Router -->|hash(user_id) % 4 = 3| S3[(Shard 3)]
```

**Pros:** Even distribution; simple  
**Cons:** Adding/removing shards requires rehashing ALL data (use consistent hashing to mitigate → [[14 - Consistent Hashing]])

### 2. Range-Based Sharding
```
Shard 0: user_id 1 - 1,000,000
Shard 1: user_id 1,000,001 - 2,000,000
Shard 2: user_id 2,000,001 - 3,000,000
```

**Pros:** Range queries are efficient (all data in a range is on one shard); easy to add shards at the end  
**Cons:** Hot spots! New users all go to the latest shard; inactive users on older shards waste space

> [!warning] Range Sharding Hotspot
> For a time-series key (created_at, auto-increment ID), range sharding concentrates ALL writes on the latest shard while older shards sit idle. Use hash sharding or a composite key.

### 3. Directory-Based Sharding
A lookup table maps keys to shards.

```
shard_table:
  user_id 1-500:      Shard A
  user_id 501-10000:  Shard B  (uneven but allows rebalancing)
  user_id 10001-...:  Shard C
```

**Pros:** Flexible; can rebalance by updating table; can move individual keys  
**Cons:** Lookup table is a bottleneck and single point of failure; must be cached aggressively

---

## Choosing the Shard Key

The shard key determines how data is distributed. A bad shard key creates hotspots.

| Criteria | Good | Bad |
|----------|------|-----|
| **Cardinality** | High (many distinct values) | Low (gender: M/F → 2 shards) |
| **Write distribution** | Even writes across shards | All writes to one shard |
| **Query pattern** | Queries need data from one shard | Scatter-gather across all shards |
| **Growth** | Scales with data growth | One shard grows disproportionately |

**Examples:**
- `user_id` → Good (high cardinality, even distribution)
- `country` → Bad (US has 10x traffic of smaller countries)
- `created_at` → Bad for writes (all new data to latest shard)
- `tenant_id` for multi-tenant SaaS → Good (each tenant's data on one shard)

---

## Cross-Shard Operations (The Hard Part)

### Cross-Shard Joins
```sql
-- User is on Shard 1, their orders are on Shard 3 (because different shard key)
SELECT u.name, o.total FROM users u JOIN orders o ON u.id = o.user_id
→ This requires scatter-gather: query all shards and merge in application
```

**Solutions:**
1. **Co-location**: Store related data on the same shard (use same shard key)
2. **Denormalize**: Duplicate data (embed user name in order record)
3. **Application-level join**: Fetch from each shard, merge in code

### Cross-Shard Transactions
ACID across shards requires distributed transactions → very expensive.

**Solutions:**
1. **Avoid cross-shard transactions** (good data model design)
2. **Two-Phase Commit (2PC)**: Distributed transaction, high latency → [[16 - Distributed Consensus]]
3. **Saga pattern**: Sequence of local transactions with compensating transactions on failure
4. **Eventual consistency**: Accept that cross-shard operations are eventually consistent

### Cross-Shard Aggregations
```sql
-- How many total users? → query all shards and sum
SELECT COUNT(*) FROM users  → scatter to all shards, sum results
```
For analytics: replicate to a data warehouse (Snowflake, BigQuery) instead.

---

## Hotspots and Solutions

### Celebrity/Whale Problem
One shard key generates far more traffic than others (Justin Bieber on Twitter, hot product on Amazon).

**Solutions:**
1. **Logical sharding**: Split the hot key into sub-keys
   - `user:bieber:shard:0`, `user:bieber:shard:1`, ..., `user:bieber:shard:9`
   - Spread writes across 10 virtual shards
2. **Caching**: Put hot data in Redis, bypass DB
3. **Read replicas for hot shards**: Add more replicas to overloaded shard

---

## Resharding (The Nightmare)

When a shard outgrows its capacity or distribution becomes uneven:

```
Initial: 4 shards → Need: 8 shards
Problem: Must move ~50% of all data to new shards
Risk: Downtime, data inconsistency during migration
```

**Resharding approaches:**
1. **Double shards**: 4 → 8 (clean split, each old shard splits into 2 new)
2. **Consistent hashing**: Minimizes data movement → [[14 - Consistent Hashing]]
3. **Logical sharding**: Shard within shard (don't change physical nodes, just logical assignment)

**Online resharding tools:** Facebook's MyRocks, Vitess (MySQL), MongoDB sharding (built-in online resharding)

---

## Sharding in Popular Databases

| Database | Sharding | Notes |
|----------|---------|-------|
| MySQL | Manual (Vitess) | Vitess is Google's open-source MySQL sharding |
| PostgreSQL | Manual (Citus) | Citus Data adds native sharding to Postgres |
| MongoDB | Built-in | Mongos router, configurable shard key |
| Cassandra | Built-in | Consistent hashing, vnodes |
| DynamoDB | Built-in (managed) | AWS manages sharding transparently |
| HBase | Built-in (Regions) | Regions automatically split |

---

## Interview Checklist

- [ ] Know 3 sharding strategies and their trade-offs
- [ ] Know how to choose a good shard key (hotspot avoidance)
- [ ] Explain cross-shard join problem and solutions (co-location, denormalization)
- [ ] Explain cross-shard transaction problem and Saga pattern
- [ ] Know the resharding problem and consistent hashing as a solution
- [ ] Know the celebrity/hotspot problem and solutions

---

## Related Notes

- [[14 - Consistent Hashing]] — the solution to hash-based resharding
- [[12 - Database Replication]] — replication works within each shard
- [[11 - SQL vs NoSQL]] — NoSQL DBs often have built-in sharding
- [[02 - CAP Theorem]] — cross-shard operations affect consistency guarantees
