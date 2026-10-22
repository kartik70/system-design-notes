---
tags: [system-design, estimation, back-of-envelope, faang, reference]
aliases: [Estimation, Back of Envelope, BOE, Capacity Planning]
created: 2026-10-22
difficulty: ⭐
type: reference
---

# Estimation Cheat Sheet

> [!tip] How to Use This
> Memorize the conversion table below. Use it to quickly estimate any system's requirements in under 5 minutes. Always round to nearest order of magnitude — precision is not the goal.

---

## Quick Conversions

### Time
| Interval | Seconds |
|----------|---------|
| 1 minute | 60 |
| 1 hour | 3,600 |
| 1 day | 86,400 ≈ **100K** |
| 1 month | 2.6M ≈ **2.5M** |
| 1 year | 31.5M ≈ **30M** |

### Request Rate
| Daily requests | Requests/sec |
|---------------|-------------|
| 1M/day | 12/sec ≈ **~10** |
| 10M/day | 120/sec ≈ **~100** |
| 100M/day | 1,200/sec ≈ **~1K** |
| 1B/day | 12,000/sec ≈ **~10K** |
| 10B/day | 120,000/sec ≈ **~100K** |

### Data Size
| Unit | Bytes |
|------|-------|
| 1 KB | 10^3 |
| 1 MB | 10^6 |
| 1 GB | 10^9 |
| 1 TB | 10^12 |
| 1 PB | 10^15 |
| 1 EB | 10^18 |

---

## Latency Reference Numbers

| Operation | Latency | Implication |
|-----------|---------|-------------|
| L1 cache | 1 ns | |
| RAM access | 100 ns | |
| Redis GET | ~0.5-1 ms | |
| DB query (simple) | 1-5 ms | |
| SSD random read | 150 μs | |
| Network same DC | 0.5 ms | |
| Network cross-DC (US-US) | 50-100 ms | Don't do sync cross-DC |
| Network US-Asia | 200-400 ms | Much too slow for sync |
| DNS lookup (cold) | 10-100 ms | Always DNS cache |

> [!warning] Cross-DC Synchronous Calls
> If cross-DC RTT is 100ms and you have 5 service calls in sequence: 500ms for one user request. Use async replication for cross-DC, not synchronous calls.

---

## Common App Estimates

### Twitter Scale
```
DAU: 300M
Tweets/day: 500M → ~6K writes/sec
Timeline loads/day: 500M × 5 = 2.5B → ~29K reads/sec
Read:Write = 5:1
Tweet storage: 300 bytes × 500M = 150 GB/day
5 years: 150 GB × 365 × 5 ≈ 274 TB
```

### Instagram Scale
```
DAU: 500M
Photos uploaded/day: 100M → ~1.2K uploads/sec
Average photo: 3 MB → 300 TB/day raw
After compression (50%): 150 TB/day
5 years: 150 TB × 365 × 5 ≈ 274 PB
```

### WhatsApp Scale
```
DAU: 500M (assume 2B users)
Messages/day: 100B → ~1.2M messages/sec
Avg message size: 100 bytes
Storage/day: 100B × 100 bytes = 10 TB/day
```

### YouTube Scale
```
DAU: 100M
Videos uploaded/minute: 500 hours
1 hour video at 1080p: ~8 GB → 500 hours = 4,000 GB = 4 TB raw/minute
After encoding to multiple qualities: ~3 TB/minute
Storage per year: 3 TB × 60 min × 24 hr × 365 ≈ 1.5 EB/year
Streams/day: 100M users × 5 videos = 500M streams → ~6K streams/sec
Avg bitrate: 4 Mbps → 6K × 4 Mbps = 24 Gbps (just to your servers, then to CDN)
```

### Google Search Scale
```
Queries/day: 8.5B → ~100K queries/sec
Average query response: 0.5 seconds
Concurrent queries: 100K × 0.5s = 50K concurrent at any moment
Web index size: ~100 PB (estimate)
```

---

## Storage Estimation Template

```
Step 1: Objects/day
  DAU × actions_per_user = objects/day

Step 2: Size per object
  Text: 100 bytes - 10 KB
  Image (thumbnail): 10-100 KB
  Image (full): 1-5 MB
  Video (1 min, 720p): ~100 MB
  
Step 3: Total storage
  objects/day × size × 365 × retention_years

Step 4: Growth buffer
  × 1.5 for metadata, indexes, redundancy
  × 3 for replication factor (3 copies)
```

---

## Network Bandwidth Template

```
Read bandwidth:
  read_QPS × avg_response_size (MB/s)
  
Write bandwidth:
  write_QPS × avg_request_size (MB/s)
  
Example (Twitter):
  Read: 29K reads/sec × 10 KB/response = 290 MB/s = 2.3 Gbps
  Write: 6K writes/sec × 1 KB/tweet = 6 MB/s = 48 Mbps
  
  Network bandwidth needed: 2.3 Gbps minimum
  (factor of 10 for headroom: need 23 Gbps capable infrastructure)
```

---

## Server Estimation

```
Concurrent requests a server can handle:
  CPU-bound: 100-500 concurrent requests per server
  I/O-bound (DB wait): 1000-10000 concurrent connections
  WebSocket: 10K-100K connections per server
  
Number of servers needed:
  servers = peak_QPS / requests_per_server_per_second
  
Example: 100K QPS, each request takes 50ms on one CPU core
  Server can handle: 1s / 50ms = 20 req/sec per core
  8-core server: 8 × 20 = 160 req/sec per server
  Servers needed: 100K / 160 ≈ 625 servers
  Add 2x headroom: ~1250 servers
```

---

## Cache Sizing

```
Working set principle (Pareto 80/20):
  80% of reads are for 20% of data
  Cache that 20% = ~80% hit rate
  
Cache size = total_data_size × 0.20
  
For Redis:
  Single Redis instance: up to 128 GB
  Redis Cluster: multiple nodes, terabytes possible
  
Cache hit rate effect:
  DB load = read_QPS × (1 - hit_rate)
  
  If read_QPS = 100K, hit_rate = 80%:
    DB sees only 20K reads/sec (vs 100K without cache)
```

---

## Database Sizing

```
Single MySQL instance limits (rule of thumb):
  Storage: up to ~2-4 TB practical (SSD)
  Write throughput: ~5K-10K writes/sec
  Read throughput: ~50K-100K reads/sec (with read replicas)
  
When to shard MySQL:
  When single instance storage > 4 TB
  OR write throughput > 10K/sec
  OR single-instance is SPOF for business-critical data

Single Cassandra node:
  Storage: up to ~10 TB per node
  Throughput: ~50K reads + 50K writes/sec
  Cluster scales linearly with nodes
```

---

## Sample Estimation Walkthrough

> [!example] Design a photo-sharing app. Estimate capacity.
> 
> "Let's assume: 500M DAU. Users upload 1 photo/day on average."
> 
> **Writes:**
> - Photos/day = 500M × 1 = 500M photos/day
> - Write QPS = 500M / 100K ≈ **5K writes/sec**
> 
> **Reads:**
> - Assume 10 photo views per user per day = 5B views/day
> - Read QPS = 5B / 100K ≈ **50K reads/sec** (10:1 ratio)
> 
> **Storage:**
> - Avg photo: 3 MB original, compressed to 500 KB for delivery
> - Storage/day = 500M × 3 MB = **1.5 PB/day** raw uploads (!)
> - After compression: 500M × 500 KB = **250 TB/day** for delivery sizes
> - 5-year total: 250 TB × 365 × 5 ≈ **450 PB**
> - → Object storage (S3), distributed at massive scale
> 
> **Bandwidth:**
> - Read bandwidth: 50K reads/sec × 500 KB = **25 GB/s = 200 Gbps**
> - → CDN is absolutely required

---

## Checklist for Estimation

- [ ] Calculate write QPS (DAU × actions / 86,400)
- [ ] Calculate read QPS (write QPS × read:write ratio)
- [ ] Calculate storage (per record × records/day × retention)
- [ ] Calculate bandwidth (QPS × avg_size)
- [ ] Estimate cache size (20% of working data)
- [ ] Estimate servers needed
- [ ] Round to nearest order of magnitude, state assumptions

---

## Related Notes

- [[Interview Framework]] — when to do estimation in the interview
- [[05 - Latency Numbers Every Engineer Should Know]] — latency reference
- [[🗺️ System Design MOC]] — quick reference table
