---
tags: [system-design, consistency, distributed-systems, fundamentals, faang]
aliases: [Eventual Consistency, Strong Consistency, Causal Consistency]
created: 2026-10-22
difficulty: ⭐⭐⭐
type: concept
---

# 03 — Consistency Patterns

> [!important] Core Idea
> Consistency isn't binary. There's a spectrum from **strong** (most consistent, highest latency) to **weak** (least consistent, lowest latency). Picking the right level is a key design decision.

---

## The Consistency Spectrum

```
STRONG ←————————————————————————————→ WEAK
Linearizability → Sequential → Causal → Eventual → Weak
(most consistent)                              (least consistent)
     ↓                                               ↓
Higher latency                              Lower latency
Lower availability                          Higher availability
```

---

## Types of Consistency

### 1. Strong Consistency (Linearizability)
After a write completes, all subsequent reads **from any node** see the write. The system behaves as if it were a single machine.

```
Client A: Write(X=5)  ——————→ OK
Client B:                        Read(X) → 5  ✓
```

**Used in:** ZooKeeper, etcd, Google Spanner (TrueTime), HBase  
**Cost:** High latency, requires quorum or consensus  
**Use when:** Leader election, distributed locks, financial transactions

---

### 2. Sequential Consistency
All operations appear to execute in some sequential order. All nodes agree on the order, but it may not be real-time order.

- Weaker than linearizability (no real-time guarantee)
- Stronger than causal (all ops, not just causally related)

---

### 3. Causal Consistency
Operations that are **causally related** appear in the same order everywhere. Concurrent operations may appear in different orders at different nodes.

```
A: Write(X=1)
B: Read(X=1) → Write(Y=2)  [caused by seeing X=1]
C: Read(Y=2)               [must see X=1 before Y=2]
D: Read(X=?)               [may see X=0 or X=1 — no causal relationship]
```

**Used in:** MongoDB causally consistent sessions, some Cassandra configurations  
**Use when:** Comments on a post (comment replies must appear after the original)

---

### 4. Eventual Consistency
Given enough time with no new writes, **all replicas will converge** to the same value. No guarantee on when.

```
Time 0:  Node A: X=5, Node B: X=3 (stale)
Time 1:  Node A: X=5, Node B: X=5 (converged)
```

**Used in:** Cassandra (default), DynamoDB, DNS, Amazon S3  
**Use when:** Social media likes/views, shopping recommendations, user profiles

> [!warning] Eventual Consistency ≠ Inconsistency
> It means the system **will** become consistent. The question is when. For many use cases, a few seconds of staleness is completely acceptable.

---

### 5. Read-Your-Writes Consistency
After a write, the **same client** always sees their own write. Other clients may still see stale data.

```
Client A: Write(X=5)
Client A: Read(X) → 5  ✓ (always)
Client B: Read(X) → 3  (may see stale data)
```

**Implementation:**
- Always route user's reads to the same replica (sticky sessions)
- Track write timestamp, only read from replicas caught up to that timestamp
- Read from primary after a write for a short time window

**Used in:** Facebook's TAO, most user-facing social apps

---

### 6. Monotonic Reads
If a client reads value V at time T, subsequent reads never return a value older than V.

```
Client A: Read(X) → 5   (version 2)
Client A: Read(X) → 3   ✗ VIOLATION (version 1 is older)
Client A: Read(X) → 5   ✓
Client A: Read(X) → 7   ✓ (version 3, newer)
```

**Implementation:** Always route client to same replica, or track vector clocks

---

### 7. Monotonic Writes
Writes from the same client are applied in order.

```
Client A: Write(X=1) then Write(X=2)
On any node: X goes from 1 → 2, never 2 → 1
```

---

## Conflict Resolution in Eventual Consistency

When two nodes receive conflicting writes, how to resolve?

### Last Write Wins (LWW)
- Use timestamp to determine winner
- Problem: Clock skew in distributed systems → wrong ordering
- Used by: Cassandra (default)

### Vector Clocks
- Track causality, not just time
- Can detect true conflicts vs causal ordering
- Used by: Amazon Dynamo, Riak

### CRDTs (Conflict-free Replicated Data Types)
- Data structures that automatically merge without conflicts
- Examples: counters, sets, maps with specific merge semantics
- Used by: Riak, Redis cluster

### Application-level conflict resolution
- Merge conflicting versions in application code
- Shopping cart example: merge both carts (union of items)

---

## Session Guarantees (Practical Consistency)

| Guarantee | Description | Implementation |
|-----------|-------------|----------------|
| Read-your-writes | See your own writes | Sticky replica or timestamp tracking |
| Monotonic reads | Values don't go backward | Sticky replica |
| Monotonic writes | Writes applied in order | Single master per key |
| Writes-follow-reads | Write after read respects causality | Vector clocks |

> [!tip] Session Guarantees in Practice
> Most user-facing systems implement **Read-your-writes + Monotonic reads** as the minimum session guarantee. This prevents jarring user experiences (e.g., posting a comment then refreshing and not seeing it).

---

## Choosing the Right Consistency Level

```
Question: Can the user tolerate seeing stale data?
  → Yes (social feed, views): Eventual consistency
  → No (bank balance): Strong consistency

Question: Does the write need to be durable before responding?
  → Yes: Synchronous replication, quorum writes
  → No: Async replication, fast writes

Question: Do I need cross-session consistency?
  → No: Read-your-writes is sufficient
  → Yes: Strong/linearizable consistency
```

---

## Interview Checklist

- [ ] Know the spectrum from linearizable to weak
- [ ] Be able to explain eventual consistency with a concrete example
- [ ] Know read-your-writes and why users need it
- [ ] Know conflict resolution strategies (LWW, vector clocks, CRDTs)
- [ ] Connect consistency choices to CAP theorem
- [ ] Give concrete database examples for each type

---

## Related Notes

- [[02 - CAP Theorem]] — the fundamental constraint
- [[12 - Database Replication]] — how replicas create consistency challenges
- [[16 - Distributed Consensus]] — mechanism to achieve strong consistency
- [[11 - SQL vs NoSQL]] — consistency guarantees per database type
