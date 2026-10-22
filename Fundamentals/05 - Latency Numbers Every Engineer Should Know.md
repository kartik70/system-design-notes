---
tags: [system-design, latency, performance, fundamentals, faang]
aliases: [Latency Numbers, Performance Numbers, Jeff Dean Numbers]
created: 2026-10-22
difficulty: ⭐
type: reference
---

# 05 — Latency Numbers Every Engineer Should Know

> [!important] Why These Matter
> These numbers let you reason about performance trade-offs. When someone says "just add a DB call", you should immediately know that's ~1ms vs an L1 cache hit at ~1ns — a million times difference.

---

## The Classic Table (2024-updated)

| Operation | Latency | Notes |
|-----------|---------|-------|
| L1 cache hit | ~1 ns | On-chip cache |
| L2 cache hit | ~4 ns | On-chip, slightly further |
| L3 cache hit | ~10-40 ns | Shared across cores |
| RAM read | ~100 ns | Main memory access |
| SSD random read | ~150 μs | ~100,000x slower than RAM |
| SSD sequential read (1MB) | ~1 ms | Great for sequential I/O |
| HDD random read | ~10 ms | ~100x slower than SSD |
| HDD sequential read (1MB) | ~20 ms | |
| Network round trip (same DC) | ~0.5 ms | Intra-datacenter |
| Redis get | ~0.5-1 ms | Network + memory |
| MySQL query (simple) | ~1-5 ms | Network + disk I/O |
| Network round trip (cross-DC) | ~50-150 ms | e.g., US East ↔ US West |
| Network round trip (global) | ~150-400 ms | e.g., US ↔ Asia |
| Mutex lock/unlock | ~25 ns | |
| System call | ~50-200 ns | Context switch |
| Send 1MB over network (1Gbps) | ~10 ms | |
| DNS lookup | ~1-100 ms | Cached vs cold |

---

## Scaled to Human Time (Analogy)

Scaling 1 ns → 1 second:

| Operation | Actual | Scaled |
|-----------|--------|--------|
| L1 cache | 1 ns | 1 second |
| L2 cache | 4 ns | 4 seconds |
| RAM | 100 ns | 1.7 minutes |
| SSD | 150 μs | ~1.5 days |
| HDD | 10 ms | ~4 months |
| Cross-DC network | 150 ms | ~5 years |
| Global round trip | 400 ms | ~12 years |

> [!tip] The Takeaway
> RAM is orders of magnitude faster than disk. Keep hot data in memory (cache). Cross-DC calls are expensive — co-locate related services.

---

## What This Means for System Design

### Rule 1: Cache > DB
```
Redis GET:    ~1 ms
MySQL query:  ~5 ms (simple), ~100ms+ (complex)
→ Cache saves 5-100x latency on reads
```

### Rule 2: In-process > Network
```
In-process function call: ~nanoseconds
Network call to another service: ~0.5-5ms
→ Minimize network hops; batch calls when possible
```

### Rule 3: Cross-DC is Expensive
```
Same DC:      ~0.5 ms
Cross-DC:     ~50-150 ms
→ Keep synchronous calls within the same DC
→ Cross-DC only for async replication or user routing
```

### Rule 4: SSD is Good Enough for Many Use Cases
```
SSD random read: ~150 μs
Great for: database primary storage, write-ahead logs
Avoid: random reads at very high QPS (prefer cache)
```

---

## Powers of Two (For Estimation)

| Power | Approximate Value | Easy Name |
|-------|------------------|-----------|
| 2^10 | 1,024 | ~1 Thousand (KB) |
| 2^20 | 1,048,576 | ~1 Million (MB) |
| 2^30 | 1,073,741,824 | ~1 Billion (GB) |
| 2^40 | ~1 Trillion | TB |
| 2^50 | ~1 Quadrillion | PB |

---

## Time Conversions

| Unit | Seconds | Requests (at 1 req/s) |
|------|---------|----------------------|
| 1 minute | 60 | 60 |
| 1 hour | 3,600 | 3,600 |
| 1 day | 86,400 | ~100,000 (10^5) |
| 1 month | ~2.6M | ~2.6M |
| 1 year | ~31.5M | ~31.5M |

> [!tip] Quick Conversion
> 1 million requests/day ≈ **12 requests/second**  
> 1 billion requests/day ≈ **12,000 requests/second** (12K RPS)

---

## Throughput vs Latency

```
Throughput = how much work in a time period (RPS, MB/s)
Latency = how long one request takes (ms, μs)

High throughput + Low latency = ideal but often in tension
  → Batching increases throughput but increases latency
  → Caching reduces latency but requires memory

At FAANG:
  → P99 latency is the metric (not average)
  → Average hides the 1% of users having terrible experience
  → Tail latency (P999) matters for large fan-out operations
```

---

## Tail Latency Amplification

> [!warning] Critical Concept for FAANG
> If a search request fans out to 100 servers, the total latency = slowest server response. Even if each server responds in P99 = 50ms, with 100 servers:
> 
> P(all 100 respond within 50ms) = 0.99^100 ≈ 37%
> 
> So 63% of user requests take longer than P99! This is why distributed systems care deeply about tail latency.

**Solutions:**
- **Hedged requests**: Send same request to 2 servers, use whichever responds first
- **Timeout and retry**: Cancel slow requests, retry elsewhere
- **Limit fan-out**: Don't fan out to hundreds of services synchronously

---

## Bandwidth Numbers

| Link | Bandwidth |
|------|-----------|
| 1Gbps NIC (datacenter) | 125 MB/s |
| 10Gbps NIC (modern servers) | 1.25 GB/s |
| 100Gbps network spine | 12.5 GB/s |
| SSD sequential | 500 MB/s - 7 GB/s (NVMe) |
| RAM bandwidth | ~50 GB/s |

---

## Interview Checklist

- [ ] Know the order of magnitude for each storage type
- [ ] Be able to estimate if a design will hit latency targets
- [ ] Know that SSD >> HDD, RAM >> SSD, L1 >> RAM
- [ ] Know cross-DC latency (~150ms) — avoid synchronous cross-DC
- [ ] Know that P99 latency matters more than average
- [ ] Know tail latency amplification in fan-out systems

---

## Related Notes

- [[01 - Scalability]] — why these numbers drive scaling decisions
- [[07 - Caching]] — the primary tool for latency reduction
- [[Estimation Cheat Sheet]] — using these numbers in interview estimates
