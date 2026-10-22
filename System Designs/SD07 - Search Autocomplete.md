---
tags: [system-design, search, autocomplete, trie, faang, problem]
aliases: [Autocomplete, Typeahead, Search Suggestions, Trie]
created: 2026-10-22
difficulty: ⭐⭐⭐
type: problem
asked-by: [Google, Amazon, LinkedIn, Twitter]
---

# SD07 — Search Autocomplete (Typeahead)

> [!tip] Interview Strategy
> Show you know the **Trie** data structure and why it's used, then address the distributed trie problem. The interviewer wants to see how you make a Trie scale to billions of queries and update suggestions in near-real-time.

---

## Requirements Clarification

### Functional Requirements
- [ ] Return top 5 search suggestions as user types
- [ ] Suggestions ranked by frequency (most searched first)
- [ ] Response time < 100ms (feels instantaneous)
- [ ] Suggestions updated with new search trends

### Non-Functional Requirements
- [ ] 10 billion searches/day
- [ ] Low latency: suggestions in < 100ms P99
- [ ] High availability

---

## Capacity Estimation

```
10B searches/day ÷ 86,400 ≈ 116,000 searches/sec
Average query: 5 characters typed → 5 keystrokes × 116K = 580,000 suggestions/sec

Data: 5 million distinct search terms, avg 20 chars each
  → ~100 MB raw, but Trie is larger due to tree structure (~500 MB RAM)
```

---

## Data Structure: Trie (Prefix Tree)

```
Trie for: ["apple", "app", "application", "apt"]

Root
 └─ 'a'
     └─ 'p'
         ├─ 'p' (count=25)
         │   ├─ 'l'
         │   │   ├─ 'e' (count=100) ← "apple"
         │   │   └─ 'i'
         │   │       └─ 'c'...  ← "application"
         │   └─ [END] (count=200) ← "app"
         └─ 't' (count=50) ← "apt"
```

**Search "ap":**
1. Traverse to "ap" node
2. DFS/BFS all descendants
3. Return top 5 by count

**Complexity:**
- Build: O(n × m) where n=terms, m=avg term length
- Search: O(p + q) where p=prefix length, q=top-k results traversal

### Optimization: Cache Top-K at Each Node
Instead of traversing children on every query, pre-cache top 5 results at each node:

```
Node "ap": top5 = ["apple": 100, "application": 80, "app": 75, ...]
Node "app": top5 = ["apple": 100, "application": 80, "app": 75, ...]
```

Now search "ap" = O(prefix_length) = near O(1) for short prefixes!

**Trade-off:** Each node stores extra data. For trie with 26 children × N levels, this uses more memory.

---

## System Architecture

```mermaid
graph TB
    User -->|type keystroke| CDN[CDN\nEdge Cache]
    CDN -->|cache miss| LB[Load Balancer]
    LB --> Trie[Trie Service\n(in-memory trie\nper node)]
    Trie --> Redis[(Redis Cache\n"ap" → [apple, application...])]
    
    Search[Search Logs] --> Kafka[Kafka\nsearch-events]
    Kafka --> Aggregator[Aggregator\nSpark/Flink]
    Aggregator -->|top queries per prefix| DB[(Trie DB\nMySQL)]
    DB --> TrieBuilder[Trie Builder\n(offline rebuild)]
    TrieBuilder -->|new trie snapshot| Trie
```

---

## Distributed Trie

A single trie can't fit in one machine's memory at Google scale (billions of distinct queries).

### Shard by First Letter
```
Shard 1: Queries starting with a-f
Shard 2: Queries starting with g-p
Shard 3: Queries starting with q-z

Router: hash(query[0]) → shard
```

**Problem:** Uneven distribution ('s' is much more common than 'x')

### Shard by Prefix Range
```
Shard 1: Prefixes a-az99...
Shard 2: Prefixes b-bm99...
(Ranges sized for equal traffic)
```

**Dynamic rebalancing:** If one prefix range becomes too hot (e.g., "cov" during COVID), split that range.

---

## Real-time Update Pipeline

```
Problem: Trie should reflect current trending queries
  "corona" shoots up in popularity → should appear in suggestions

Option 1: Rebuild trie offline every hour
  Simple, consistent
  1 hour lag (acceptable for most cases)

Option 2: Update trie incrementally
  Aggregator: every 10 minutes, compute delta of search frequencies
  Apply updates to trie nodes
  More complex, near-real-time

For interview: recommend hourly rebuild for simplicity, mention streaming option
```

### Data Pipeline for Frequency Aggregation

```
1. Every search logged to Kafka
2. Flink/Spark job aggregates:
   - Window: 1 week of searches
   - Group by: normalized query (lowercase, trimmed)
   - Count occurrences
3. Top N queries per prefix computed
4. Written to Trie DB (MySQL)
5. Trie Service reloads snapshot (nightly or hourly)
```

---

## Caching Strategy

Two-level cache to minimize trie traversal:

```
Level 1: Browser cache (localStorage)
  Cache last 100 prefixes the user searched
  Instant for repeated searches

Level 2: CDN edge cache
  Cache top prefixes globally
  "ap" → cached at edge, 99% of users hit edge
  TTL: 5 minutes (balance freshness vs cache efficiency)

Level 3: Redis cache (centralized)
  All prefixes up to 5 chars cached
  5 chars = 26^5 = ~12M possible, but only popular ones stored
  Cache key: "suggest:ap" → ["apple", "application", "apt"]

Level 4: In-memory Trie
  Consulted only on cache miss
```

---

## Handling Special Cases

### Different Languages / Unicode
```
Chinese/Japanese: character-based trie (not letter-based)
Arabic: right-to-left text
Emojis: stored as Unicode code points

Solution: Trie nodes store Unicode characters (not just ASCII)
Sharding: by Unicode range (CJK = separate shard)
```

### Filtering Offensive Suggestions
```
Blocklist: Set of banned terms
  Before returning suggestions: filter out any in blocklist
  Blocklist stored in Redis as Set: O(1) lookup
  Updated by content moderation team
```

### Personalized Suggestions
```
Mix global top-K with user's personal search history
  Global: "app" → ["apple", "application"]
  User history: user searched "applebees" last week
  Personalized: "app" → ["applebees", "apple", "application"]

Implementation:
  Personal trie per user (stored in user session / Redis)
  Merge with global trie on read
  Weight: personal_score × α + global_score × (1-α)
```

---

## Interview Checklist

- [ ] Explain Trie data structure with example
- [ ] Optimize: cache top-K at each node (O(1) lookup vs DFS)
- [ ] Explain two-level caching (browser, CDN, Redis, Trie)
- [ ] Explain distributed trie and sharding strategy
- [ ] Explain update pipeline (Kafka → Flink → Trie rebuild)
- [ ] Discuss filtering (blocklist), personalization, multi-language

---

## Related Notes

- [[07 - Caching]] — multi-level caching strategy
- [[09 - CDN]] — edge caching for common prefixes
- [[08 - Message Queues]] — Kafka for search event streaming
- [[15 - Indexing Strategies]] — Trie vs inverted index for search
