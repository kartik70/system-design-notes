---
tags: [system-design, indexing, databases, b-tree, lsm-tree, faang]
aliases: [Index, B-tree, LSM Tree, Inverted Index, Bloom Filter Index]
created: 2026-10-22
difficulty: ⭐⭐⭐
type: concept
---

# 15 — Indexing Strategies

> [!important] Core Idea
> Indexes speed up reads by trading write performance and storage. Choosing the right index type is critical for query performance. At FAANG, understanding B-tree vs LSM tree trade-offs explains why different databases are used for different workloads.

---

## B-Tree Index (OLTP)

Used by: PostgreSQL, MySQL, Oracle, SQLite

```
            [40 | 60]
           /    |    \
    [10|20|30] [45|50] [70|80|90]
    /  |  |  \
  leaves: actual data rows
```

**Properties:**
- Balanced tree; O(log N) reads and writes
- Each node is a disk page (typically 4KB or 16KB)
- All leaves at same depth (self-balancing)
- Leaf nodes often contain actual data (clustered index) or pointers to rows

**Strengths:**
- Excellent for range queries (`WHERE age BETWEEN 20 AND 30`)
- Good for equality queries (`WHERE id = 123`)
- Sorted order → ORDER BY on indexed column is free
- ~O(log N) for both reads and writes

**Weaknesses:**
- Write amplification: A single write may cause multiple page writes (tree rebalancing)
- Not optimal for high write throughput
- Large write amplification on SSDs reduces SSD lifespan

---

## LSM Tree (Log-Structured Merge Tree)

Used by: Cassandra, HBase, LevelDB, RocksDB, InfluxDB, ScyllaDB

**Design:** Optimized for high write throughput by making all writes sequential.

```mermaid
graph TB
    Write[Write: key=k, value=v] --> WAL[WAL\nWrite-Ahead Log\n(durability)]
    Write --> MemTable[MemTable\nin-memory sorted structure]
    MemTable -->|flush when full| L0[L0: SSTables\n(immutable sorted files)]
    L0 -->|compaction| L1[L1: Larger SSTables]
    L1 -->|compaction| L2[L2: Even larger SSTables]
    
    Read[Read: key=k] --> MemTable
    MemTable -->|if not found| L0
    L0 -->|if not found| L1
```

**Write path:**
1. Write to WAL (sequential write → durable)
2. Write to MemTable (in-memory sorted structure, e.g., red-black tree)
3. When MemTable full, flush to disk as SSTable (immutable, sorted file) — L0
4. Background compaction merges SSTables, removes tombstones (deletes)

**Read path:**
1. Check MemTable (most recent data)
2. Check L0 SSTables (newest first)
3. Check L1, L2... (increasingly larger, older)

**Key insight:** Writes are always sequential (WAL + append-only MemTable flush) → no random I/O → very fast writes

**Bloom filters:** Each SSTable has a Bloom filter to skip SSTables that definitely don't have the key → reduces unnecessary I/O → [[19 - Bloom Filters]]

### B-Tree vs LSM Tree

| Aspect | B-Tree | LSM Tree |
|--------|--------|---------|
| Write speed | Good (random I/O) | Excellent (sequential I/O) |
| Read speed | Excellent | Good (may check multiple levels) |
| Write amplification | Medium | High (compaction) |
| Read amplification | Low | Medium (multiple files) |
| Space amplification | Low | High (until compaction) |
| Use case | Read-heavy OLTP | Write-heavy (Cassandra, HBase) |
| Compaction | Not needed | Required (background) |

---

## Types of Indexes

### Primary Index (Clustered Index)
- Rows stored in index order (physically sorted by key)
- One per table (the table IS the index)
- MySQL InnoDB: clustered on primary key
- Range scans are very efficient

### Secondary Index
- Separate structure pointing to primary key (or row pointer)
- Multiple per table allowed
- Lookup: secondary index → primary key → row (2 lookups for uncovered query)

### Composite Index
```sql
CREATE INDEX idx_user_date ON orders (user_id, created_at);

-- Uses index: (leftmost prefix rule)
SELECT * FROM orders WHERE user_id = 123;
SELECT * FROM orders WHERE user_id = 123 AND created_at > '2024-01-01';

-- Does NOT use index:
SELECT * FROM orders WHERE created_at > '2024-01-01';  -- missing leftmost column
```

> [!tip] Composite Index Column Order
> Put most selective column first (unless range query at end). Leftmost prefix rule: index used if query filters from left.

### Covering Index
Index contains all columns needed by query. No need to fetch actual row.

```sql
-- Table: orders (id, user_id, status, created_at, total)
CREATE INDEX idx_covering ON orders (user_id, status, total);

-- This query is covered by the index (no row lookup needed):
SELECT user_id, status, total FROM orders WHERE user_id = 123;
```

### Partial Index
Index only on rows matching a condition.

```sql
-- Only index active users (most queries are on active users)
CREATE INDEX idx_active_users ON users (email) WHERE status = 'active';

→ Smaller, faster index; only useful for queries on active users
```

---

## Inverted Index (Full-Text Search)

Used by: Elasticsearch, Lucene, Solr, PostgreSQL (GIN)

```
Document 1: "the quick brown fox"
Document 2: "the lazy brown dog"

Inverted index:
  "the"   → [Doc 1, Doc 2]
  "quick" → [Doc 1]
  "brown" → [Doc 1, Doc 2]
  "fox"   → [Doc 1]
  "lazy"  → [Doc 2]
  "dog"   → [Doc 2]

Query: "brown fox" → Docs containing both "brown" AND "fox" → [Doc 1]
```

This is how full-text search works. Each term maps to a list of documents containing it.

**TF-IDF**: Terms score based on frequency in document vs frequency across all documents (rare terms = more relevant).

---

## Geospatial Indexing

For location-based queries ("find restaurants within 5km"):

### Geohash
```
Encode (latitude, longitude) as a short string
  (40.7128, -74.0060) → "dr5r7"  (New York City)
  
Nearby locations share a common prefix:
  dr5r7 = specific block
  dr5r  = neighborhood
  dr5   = borough
  
Index on geohash string → prefix search = area search
```

### Quadtree
Recursively divide 2D space into 4 quadrants until each cell has few points.

```
Level 0: entire world
Level 1: 4 quadrants (NE, NW, SE, SW)
Level 2: 16 quadrants
...
Leaf: small cell with list of points
```

**Used by:** Google Maps, Twitter's geosearch

### S2 Library (Google's approach)
Maps the sphere to a Hilbert curve (1D). Nearby points on sphere → nearby on curve → B-tree index works.

---

## Index Trade-offs

| Adding Index | Benefit | Cost |
|-------------|---------|------|
| Read queries | O(log N) vs O(N) | Write overhead: update index on every INSERT/UPDATE/DELETE |
| Range queries | Fast range scans | Additional storage |
| Joins | Faster join execution | More maintenance during writes |

> [!warning] Too Many Indexes
> A table with 20 indexes has 20x write overhead. Every INSERT must update all indexes. For write-heavy tables, fewer indexes are better. Drop unused indexes.

```sql
-- Find unused indexes in PostgreSQL:
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0  -- never used!
ORDER BY schemaname, tablename;
```

---

## Interview Checklist

- [ ] Know B-tree structure and why it's good for OLTP reads
- [ ] Know LSM tree write path and why it's fast for writes
- [ ] Compare B-tree vs LSM tree trade-offs
- [ ] Know composite index and leftmost prefix rule
- [ ] Know covering index and why it eliminates row lookups
- [ ] Explain inverted index for full-text search
- [ ] Know geospatial indexing (Geohash or Quadtree)
- [ ] Know the cost of indexes on write-heavy tables

---

## Related Notes

- [[11 - SQL vs NoSQL]] — B-tree in SQL, LSM tree in Cassandra/RocksDB
- [[19 - Bloom Filters]] — used in LSM tree to skip SSTables on reads
- [[SD07 - Search Autocomplete]] — inverted index via Elasticsearch
- [[SD06 - Uber]] — geospatial indexing for location-based matching
