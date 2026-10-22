---
tags: [system-design, interview, q-and-a, faang]
aliases: [Common Questions, Q&A, Interview Questions]
created: 2026-10-22
difficulty: ⭐⭐⭐
type: reference
---

# Common Questions & Answers

> [!tip] How to Use
> These are frequently asked follow-up questions in FAANG system design interviews. Study the structure of each answer — not just the content.

---

## Architecture & Design

### Q: When would you use microservices vs monolith?

**Answer:**
- **Start with monolith**: Simpler to develop, deploy, debug; lower overhead; fine until team > 10 engineers or clear service boundaries emerge
- **Migrate to microservices when**: Services have different scaling needs, different release cadences, or different tech stacks; team is large enough that monolith creates organizational bottleneck (Conway's Law)
- **Costs of microservices**: Network latency between services, distributed tracing complexity, distributed transactions, service discovery, data consistency

> **Rule of thumb**: If you can deploy the whole thing in < 5 minutes and the team is < 15 people, monolith is probably fine.

---

### Q: How do you design for horizontal scalability from day 1?

- **Stateless services**: Session state in Redis, not in-process
- **Externalizing state**: DB, cache, queue — no local state that can't be lost on restart
- **Idempotent APIs**: Same request twice → same result (safe retries)
- **Async processing**: Events/queues for heavy work
- **No shared mutable state** between service instances

---

### Q: How would you handle a traffic spike 10x normal?

1. **Auto-scaling**: Pre-configure ASG (AWS) or HPA (K8s) with headroom up to 10x
2. **Pre-warm cache**: Before known events (game launch, Super Bowl), pre-populate cache with expected hot data
3. **Rate limiting**: Throttle non-essential requests at API gateway
4. **Circuit breaker**: Fast-fail expensive operations under load
5. **Queue-based load leveling**: Async heavy processing, accept work via queue
6. **Graceful degradation**: Disable non-critical features (analytics, recommendations)
7. **CDN**: Serve static/cacheable content from edge, not origin

---

## Databases

### Q: When would you use NoSQL over SQL?

Use NoSQL when:
- **Horizontal scalability required**: can't shard SQL fast enough (Cassandra, DynamoDB)
- **Schema flexibility**: data structure changes frequently (MongoDB)
- **Specific access patterns**: key-value (Redis), time-series (Cassandra), graph (Neo4j)
- **Massive write throughput**: Cassandra writes are O(1) vs B-tree overhead in SQL

Use SQL when:
- Complex JOINs required
- ACID transactions across multiple rows/tables
- Structured, normalized data with foreign key integrity
- Report-style queries with GROUP BY, aggregations

---

### Q: How do you handle a hot key in a cache or database?

**Hot key problem**: One key receives disproportionate traffic (celebrity tweet, viral product).

**Solutions:**
1. **Local caching**: In-process cache on each app server; reduces Redis load
2. **Key splitting**: `hotkey → hotkey:shard:0, hotkey:shard:1, ...hotkey:shard:N` — read randomly from one of N copies
3. **Read replicas**: For DB, route hot key reads to read replicas
4. **Async cache refresh**: Refresh before TTL expiry (no stampede)
5. **CDN**: If content is static/cacheable, push to CDN edge

---

### Q: Explain the differences between optimistic and pessimistic locking.

**Pessimistic locking**: Lock the record on read. No other transaction can modify until lock released.
```sql
SELECT * FROM orders WHERE id=1 FOR UPDATE;  -- locks row
-- do work
UPDATE orders SET status='processed' WHERE id=1;
COMMIT;  -- lock released
```
- Good when contention is high (conflicts are likely)
- Bad for performance: long locks block other transactions

**Optimistic locking**: No lock on read. On write, check if record changed since you read it.
```sql
-- Read: version=5
SELECT *, version FROM orders WHERE id=1;

-- Write: only succeed if version unchanged
UPDATE orders SET status='processed', version=6 
WHERE id=1 AND version=5;
-- If 0 rows affected: conflict! Retry.
```
- Good when contention is low (conflicts are rare)
- Better performance: no blocking
- Requires retry logic on conflict

**In system design**: Optimistic concurrency is the default for distributed systems (ETags in REST APIs, vector clocks in Dynamo, MVCC in Postgres).

---

## Caching

### Q: How do you handle cache invalidation?

"There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

**Strategies:**
1. **TTL (Time-To-Live)**: Simplest. Accept stale data for TTL duration.
2. **Write-through**: Update cache on every write. Strong consistency, higher write latency.
3. **Event-driven invalidation**: Write publishes invalidation event → cache deletes/updates the key
4. **Version-based**: Include version in cache key (`user:123:v5`). Bump version to invalidate.
5. **Immutable URLs**: Content-addressed cache keys (hash of content = key). Never invalidate, just create new.

**CDN invalidation**: Most expensive. CDN API to purge specific paths. Takes 30s-5min. Use sparingly.

---

### Q: What is cache penetration and how do you prevent it?

**Cache penetration**: Repeated queries for keys that don't exist in cache OR database. Cache misses every time, DB queried every time.

**Causes**: Typos, probing attacks, deleted data still being queried.

**Solutions:**
1. **Cache null values**: `SET user:9999999 "NULL" TTL=60s` — second request hits cache, gets "NULL", doesn't hit DB
2. **Bloom filter**: Check before querying — if bloom filter says "definitely NOT in set", return 404 without DB query → [[19 - Bloom Filters]]

---

## Consistency and Distributed Systems

### Q: What is the difference between consistency and availability in the context of distributed systems?

→ See [[02 - CAP Theorem]] for full explanation.

**Short answer**: In a distributed system with network partitions:
- **Consistency (CP)**: All nodes see the same data at the same time. May return error if data might be stale. (ZooKeeper, HBase)
- **Availability (AP)**: System always responds, even if data might be stale. Never returns error due to consistency. (Cassandra, DynamoDB)

Most systems don't need 100% strong consistency — eventual consistency + read-your-writes is sufficient for social apps. Financial systems require strong consistency.

---

### Q: How does a distributed database handle a network partition?

Depends on the system's CAP choice:

**CP system (ZooKeeper, HBase)**:
- Majority partition stays available (has quorum)
- Minority partition rejects requests (no quorum)
- After partition heals: minority catches up from majority

**AP system (Cassandra)**:
- Both partitions continue serving requests
- After partition heals: conflict resolution (last-write-wins, vector clocks)
- Temporary inconsistency is accepted

---

### Q: What is an idempotent operation? Why does it matter?

**Idempotent**: Applying the same operation multiple times has the same effect as applying it once.

```
Idempotent: 
  PUT /users/123 {name: "Alice"} → safe to retry; result always the same
  DELETE /messages/456 → deleting an already-deleted message = no-op

Non-idempotent:
  POST /payments {amount: $100} → if retried, charges twice!
  
Solution for non-idempotent: Idempotency keys
  POST /payments {amount: $100, idempotency_key: "uuid-123"}
  Server: if uuid-123 seen before → return cached result, don't charge again
```

**Why it matters**: Network failures cause retries. If operations aren't idempotent, retries cause duplicate processing (double charges, duplicate messages). Design all mutation APIs to be idempotent.

---

## Messaging & Async

### Q: When would you use a message queue vs synchronous API calls?

**Use synchronous API** when:
- Client needs immediate response (user is waiting)
- Operation is fast (< 100ms)
- No durable processing required (can retry from client)

**Use message queue** when:
- Operation is slow (video encoding, email sending, ML inference)
- Decouple producer from consumer scaling
- Need retry logic with backoff
- Need to handle traffic bursts (queue absorbs spike)
- Need to fan-out to multiple consumers

**Example**: User uploads photo → API immediately returns "Upload queued" (202 Accepted) → Queue → Resize worker, CDN warmer, face detection worker (all independent consumers)

---

### Q: Kafka vs RabbitMQ — when would you use each?

| | Kafka | RabbitMQ |
|--|-------|---------|
| **Use for** | High-throughput event streaming, replay-ability, event sourcing | Traditional message brokering, complex routing |
| **Retention** | Days to forever (log-based) | Until consumed (queue-based) |
| **Ordering** | Partition-level | Queue-level |
| **Throughput** | Millions/sec | Hundreds of thousands/sec |
| **Replay** | Yes (seek to any offset) | No |
| **Complex routing** | No | Yes (exchanges, bindings, DLQ routing) |

**Kafka when**: You need replay, audit log, event sourcing, high throughput  
**RabbitMQ when**: You need complex routing, priority queues, smaller scale

---

## Performance

### Q: How do you debug high latency in a distributed system?

1. **Check metrics**: Is it P50, P95, or P99 degraded? (P99 issue = worst requests; P50 issue = widespread)
2. **Distributed tracing**: Trace individual requests (Jaeger, Zipkin) — which service/span is slow?
3. **Check queue depths**: Is a consumer backed up? Kafka lag, SQS message count
4. **Check DB**: Slow query log, `EXPLAIN` the slow queries
5. **Check cache hit rate**: If hit rate dropped, DB is overloaded
6. **Check downstream dependencies**: Which service call takes most time?
7. **Check GC pauses**: JVM GC pauses cause P99 spikes

---

### Q: What is tail latency and why does it matter?

**Tail latency**: P99 or P999 latency — the worst 1% or 0.1% of requests.

**Why it matters for FAANG scale**:
- With 100K requests/sec, P99 = 1,000 slow requests/sec (1000 unhappy users/sec)
- Fan-out amplification: If a page request fans out to 100 backend services, page latency = SLOWEST service response. P99 of composite = 1 - (0.99^100) ≈ 63% of page requests take P99 time or longer.

**Mitigation**:
- Hedged requests: Send same request to 2 servers, use first response
- Timeout + retry: Cancel slow requests, retry elsewhere (with idempotent operations)
- Reduce fan-out: Aggregate results where possible

---

## Interview Checklist

- [ ] Can explain microservices vs monolith trade-offs
- [ ] Can explain pessimistic vs optimistic locking
- [ ] Can explain cache invalidation strategies (5 types)
- [ ] Can explain idempotency and why it matters for retries
- [ ] Can explain tail latency and fan-out amplification
- [ ] Can justify Kafka vs RabbitMQ for a scenario
- [ ] Can explain hot key problem and solutions
- [ ] Can explain cache penetration and prevention

---

## Related Notes

- [[02 - CAP Theorem]] — consistency vs availability
- [[07 - Caching]] — cache invalidation in depth
- [[08 - Message Queues]] — Kafka vs RabbitMQ deep dive
- [[Interview Framework]] — full interview structure
