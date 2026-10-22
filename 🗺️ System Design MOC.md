---
tags: [system-design, moc, faang, index]
aliases: [MOC, Index, Home]
created: 2026-10-22
type: moc
---

# 🗺️ System Design — Map of Content

> [!tip] How to Use These Notes
> These notes are structured for **FAANG interviews at the 5+ year experience level**. Each note has deep trade-off analysis, Mermaid diagrams, and interview Q&A sections. Use the checklists interactively in Obsidian.
> 
> **Recommended study order:** Fundamentals → Core Components → Databases → Distributed Systems → System Designs → Interview Prep

---

## 📐 Fundamentals

| Note | Key Concepts | Difficulty |
|------|-------------|-----------|
| [[01 - Scalability]] | Horizontal vs Vertical, Stateless, Auto-scaling | ⭐⭐ |
| [[02 - CAP Theorem]] | CP vs AP, PACELC, Real-world systems | ⭐⭐⭐ |
| [[03 - Consistency Patterns]] | Strong, Eventual, Causal, Session | ⭐⭐⭐ |
| [[04 - Availability & Reliability]] | SLA/SLO/SLI, Circuit Breaker, Bulkhead | ⭐⭐⭐ |
| [[05 - Latency Numbers Every Engineer Should Know]] | L1/L2 cache, Disk I/O, Network | ⭐ |

---

## ⚙️ Core Components

| Note | Key Concepts | Difficulty |
|------|-------------|-----------|
| [[06 - Load Balancers]] | L4 vs L7, Algorithms, Health Checks | ⭐⭐ |
| [[07 - Caching]] | Cache strategies, Eviction, Redis, Thundering Herd | ⭐⭐⭐ |
| [[08 - Message Queues]] | Kafka, Pub-Sub, Delivery Semantics | ⭐⭐⭐ |
| [[09 - CDN]] | Push vs Pull, Edge Caching, Invalidation | ⭐⭐ |
| [[10 - API Design]] | REST, gRPC, GraphQL, WebSockets | ⭐⭐⭐ |

---

## 🗄️ Databases

| Note | Key Concepts | Difficulty |
|------|-------------|-----------|
| [[11 - SQL vs NoSQL]] | ACID vs BASE, NoSQL types, Polyglot | ⭐⭐ |
| [[12 - Database Replication]] | Leader-Follower, Multi-leader, Leaderless | ⭐⭐⭐ |
| [[13 - Database Sharding]] | Hash/Range/Directory, Hotspots, Resharding | ⭐⭐⭐ |
| [[14 - Consistent Hashing]] | Ring, Virtual Nodes, Real-world usage | ⭐⭐⭐ |
| [[15 - Indexing Strategies]] | B-tree, LSM, Bloom Filters, Geospatial | ⭐⭐⭐ |

---

## 🌐 Distributed Systems

| Note | Key Concepts | Difficulty |
|------|-------------|-----------|
| [[16 - Distributed Consensus]] | 2PC, Raft, Paxos, etcd | ⭐⭐⭐⭐ |
| [[17 - Leader Election]] | Bully, Raft, Zookeeper, Split Brain | ⭐⭐⭐ |
| [[18 - Rate Limiting]] | Token Bucket, Leaky Bucket, Sliding Window | ⭐⭐⭐ |
| [[19 - Bloom Filters]] | Hash Functions, False Positives, Use Cases | ⭐⭐ |
| [[20 - Service Discovery & Health Checks]] | Consul, etcd, Service Mesh, Sidecar | ⭐⭐⭐ |

---

## 🏗️ System Design Problems

| Problem | Difficulty | Key Challenge | Asked By |
|---------|-----------|---------------|----------|
| [[SD01 - URL Shortener]] | ⭐⭐ | Base62 encoding, Cache, Analytics | All FAANGs |
| [[SD02 - Chat System (WhatsApp)]] | ⭐⭐⭐ | WebSocket, HBase, Presence | Meta, Google |
| [[SD03 - Instagram]] | ⭐⭐⭐ | Fan-out, CDN, Feed generation | Meta |
| [[SD04 - YouTube]] | ⭐⭐⭐ | Transcoding pipeline, CDN, Streaming | Google, Netflix |
| [[SD05 - Twitter]] | ⭐⭐⭐ | Fan-out on write vs read, Trending | Twitter/X |
| [[SD06 - Uber]] | ⭐⭐⭐⭐ | Geospatial index, Matching, ETA | Uber, Lyft |
| [[SD07 - Search Autocomplete]] | ⭐⭐⭐ | Distributed Trie, Top-K, Real-time | Google, Amazon |
| [[SD08 - Web Crawler]] | ⭐⭐⭐ | URL Frontier, Politeness, Dedup | Google |
| [[SD09 - Notification System]] | ⭐⭐ | Multi-channel, Deduplication, Throttle | All FAANGs |
| [[SD10 - Google Drive]] | ⭐⭐⭐ | Chunking, Sync, Conflict Resolution | Google |
| [[SD11 - News Feed (Facebook)]] | ⭐⭐⭐⭐ | Hybrid fan-out, Ranking, Real-time | Meta |
| [[SD12 - Netflix]] | ⭐⭐⭐⭐ | Open Connect CDN, Transcoding, Rec. | Netflix |

---

## 🎯 Interview Preparation

| Note | Purpose |
|------|---------|
| [[Interview Framework]] | Step-by-step interview structure |
| [[Estimation Cheat Sheet]] | Quick reference for capacity estimates |
| [[Common Questions & Answers]] | Curated Q&A for FAANG interviews |

---

## 🔑 Key Trade-offs Quick Reference

```
Consistency vs Availability → CAP Theorem
Latency vs Throughput → Caching, Async
Read vs Write optimization → Indexes, Replication
Monolith vs Microservices → Team size, Scale
Push vs Pull (Fan-out) → Celebrity problem in social networks
Strong vs Eventual consistency → Depends on business requirements
```

---

## 📊 Estimation Quick Reference

| Metric | Value |
|--------|-------|
| 1 million requests/day | ~12 req/sec |
| 1 billion requests/day | ~12,000 req/sec |
| 1 KB per message × 1B messages | ~1 TB/day |
| SSD read | ~150 μs |
| Network round trip (same DC) | ~500 μs |
| Network round trip (cross-DC) | ~150 ms |

→ Full reference: [[Estimation Cheat Sheet]]

---

## 🏷️ Tags Index

#scalability #caching #databases #distributed-systems #consistency #availability #kafka #redis #system-design #faang #interview
