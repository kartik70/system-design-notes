---
tags: [system-design, bloom-filter, probabilistic, distributed-systems, faang]
aliases: [Bloom Filter, Probabilistic Data Structure]
created: 2026-10-22
difficulty: ⭐⭐
type: concept
---

# 19 — Bloom Filters

> [!important] Core Idea
> A Bloom filter is a space-efficient probabilistic data structure that answers "Is this element in the set?" with **no false negatives** but **possible false positives**. "Definitely NOT in set" or "Probably in set."

---

## How It Works

```
Bit array: [0, 0, 0, 0, 0, 0, 0, 0]  (size m bits)
Hash functions: h1, h2, h3  (k hash functions)

ADD "apple":
  h1("apple") = 2 → set bit[2] = 1
  h2("apple") = 5 → set bit[5] = 1
  h3("apple") = 7 → set bit[7] = 1
  Array: [0, 0, 1, 0, 0, 1, 0, 1]

ADD "banana":
  h1("banana") = 1, h2("banana") = 4, h3("banana") = 7
  Array: [0, 1, 1, 0, 1, 1, 0, 1]

CHECK "apple":
  h1("apple")=2→1, h2("apple")=5→1, h3("apple")=7→1
  All bits set → "Probably in set" ✓

CHECK "cherry":
  h1("cherry")=3→0 → "Definitely NOT in set" ✓

CHECK "mango":
  h1("mango")=1→1, h2("mango")=4→1, h3("mango")=7→1
  All bits set → "Probably in set" ✗ FALSE POSITIVE!
  (These bits were set by "apple" and "banana", not "mango")
```

---

## Properties

| Property | Bloom Filter |
|----------|-------------|
| False positives | Possible (tunable) |
| False negatives | Never! |
| Delete elements | Not supported (standard) |
| Space | O(m) bits regardless of elements |
| Lookup time | O(k) hash computations |
| Insertion time | O(k) hash computations |

### False Positive Rate
```
m = number of bits
n = number of elements inserted
k = number of hash functions

Optimal k = (m/n) × ln(2)

False positive rate ≈ (1 - e^(-kn/m))^k

Example: m=1000 bits, n=100 elements, k=7
  False positive rate ≈ 0.8% (very low!)
  
With just 10 bits/element, FP rate < 1%
(Compare: 10 bits vs. storing the element itself ~100+ bits)
```

---

## Use Cases

### 1. Cache Penetration Prevention
```
Problem: Requests for non-existent keys miss cache AND miss DB → DB hammered
Solution: Bloom filter of all existing IDs

Request for user:99999999:
  Bloom filter: "Definitely NOT in set" → Return 404 immediately
  → DB never queried!
```

### 2. Web Crawler — URL Deduplication
```
Crawled billions of URLs — have we seen this URL before?
Storing all URLs in a set: ~100 bytes × 1 billion = 100 GB
Bloom filter: ~10 bits/URL = 1.25 GB (80x smaller!)

False positive = occasionally skip a valid URL = acceptable
→ [[SD08 - Web Crawler]]
```

### 3. LSM Tree SSTable Skip
```
Before reading an SSTable for key K:
  Check SSTable's bloom filter
  "Definitely NOT in set" → skip this SSTable entirely
  → Massive reduction in disk I/O for reads
Used by: RocksDB, Cassandra, HBase → [[15 - Indexing Strategies]]
```

### 4. Distributed Systems — Node Membership
```
Is node X part of this cluster?
Bloom filter of all cluster members → fast membership check
```

### 5. Database Systems — Join Acceleration
```
Before streaming table B to join with table A:
  Build bloom filter of A's join keys
  Filter B: only send rows whose join key "probably in" A
  → Reduce network transfer for distributed joins
```

### 6. Spam/Fraud Detection
```
Is this email address on our blacklist?
Blacklist has 100 million addresses
Bloom filter: O(1) lookup, tiny memory
```

---

## Limitations

### No Deletions
Once a bit is set, it can't be unset without potentially affecting other elements.

**Solutions:**
- **Counting Bloom Filter**: Each bit is a counter. Increment on add, decrement on delete. Allows deletions but uses more space (4 bits per counter instead of 1 bit).
- **Cuckoo Filter**: Supports deletions, similar space efficiency to Bloom filter.

### False Positive Rate Grows with More Elements
As more elements added, more bits set, FP rate rises. Must resize (rebuild) if too many elements.

**Scalable Bloom Filter**: Dynamically adds new filters as capacity is reached.

---

## Quick Implementation

```python
import hashlib

class BloomFilter:
    def __init__(self, size=1000, num_hashes=5):
        self.size = size
        self.num_hashes = num_hashes
        self.bit_array = bytearray(size)
    
    def _hashes(self, item):
        item = item.encode() if isinstance(item, str) else item
        for i in range(self.num_hashes):
            h = int(hashlib.md5(item + i.to_bytes(4, 'big')).hexdigest(), 16)
            yield h % self.size
    
    def add(self, item):
        for h in self._hashes(item):
            self.bit_array[h] = 1
    
    def check(self, item):
        return all(self.bit_array[h] for h in self._hashes(item))
        # Returns True = "probably in set"
        # Returns False = "definitely not in set"
```

---

## Interview Checklist

- [ ] Explain Bloom filter: bit array + k hash functions
- [ ] Know: no false negatives, possible false positives
- [ ] Know: cannot delete (unless counting variant)
- [ ] Know the false positive rate formula conceptually
- [ ] Know 3+ real use cases (cache penetration, web crawler, LSM tree)
- [ ] Know Counting Bloom Filter as deletion solution
- [ ] Know the space savings vs regular set

---

## Related Notes

- [[15 - Indexing Strategies]] — Bloom filters in LSM tree SSTables
- [[07 - Caching]] — Bloom filter to prevent cache penetration
- [[SD08 - Web Crawler]] — URL deduplication
- [[20 - Service Discovery & Health Checks]] — membership detection
