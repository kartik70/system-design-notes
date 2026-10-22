# System Design Notes — FAANG Interview Prep

> Comprehensive system design notes based on Gaurav Sen's System Design playlist, structured for **FAANG interviews at 5+ years experience level**. Optimized for Obsidian.

## How to Open in Obsidian

1. Clone this repo: `git clone <repo-url>`
2. Open Obsidian → **Open folder as vault** → select the cloned directory
3. Start from `🗺️ System Design MOC.md` (the index/home page)
4. Enable Obsidian plugins for best experience:
   - **Graph View**: See how all notes connect
   - **Dataview**: Query notes by difficulty/type
   - **Mermaid**: All diagrams render automatically (built-in)

## Structure

```
📁 Fundamentals/          ← Start here: CAP, Scalability, Consistency
📁 Core Components/       ← Load Balancers, Caching, Kafka, CDN, API Design
📁 Databases/             ← SQL vs NoSQL, Replication, Sharding, Consistent Hashing
📁 Distributed Systems/   ← Raft, Rate Limiting, Bloom Filters, Service Discovery
📁 System Designs/        ← 12 complete FAANG system design problems
📁 Interview Prep/        ← Framework, Estimation Cheat Sheet, Q&A
🗺️ System Design MOC.md   ← Main index/home (start here)
```

## Coverage

### 20 Core Topics
| Category | Topics |
|----------|--------|
| Fundamentals | Scalability, CAP Theorem, Consistency Patterns, Availability, Latency Numbers |
| Core Components | Load Balancers, Caching, Message Queues, CDN, API Design |
| Databases | SQL vs NoSQL, Replication, Sharding, Consistent Hashing, Indexing |
| Distributed Systems | Distributed Consensus (Raft), Leader Election, Rate Limiting, Bloom Filters, Service Discovery |

### 12 System Design Problems
| Problem | Key Concepts |
|---------|-------------|
| URL Shortener | Base62, Cache-aside, Analytics |
| Chat System (WhatsApp) | WebSocket, HBase, Presence, E2EE |
| Instagram | Fan-out, CDN, Feed generation |
| YouTube | Transcoding pipeline, HLS/DASH, CDN |
| Twitter | Hybrid fan-out (the celebrity problem) |
| Uber | Geospatial indexing (Geohash/H3), Matching |
| Search Autocomplete | Distributed Trie, Top-K, Real-time updates |
| Web Crawler | URL Frontier, Bloom filter dedup, Politeness |
| Notification System | Multi-channel, Dedup, Throttling |
| Google Drive | File chunking, Delta sync, Conflict resolution |
| News Feed (Facebook) | Hybrid fan-out, ML Ranking, Real-time |
| Netflix | Open Connect CDN, Per-title encoding |

## Features

- **Interactive checklists** — mark off topics as you review them
- **Mermaid diagrams** — system architecture diagrams render in Obsidian
- **Callouts** — tips, warnings, examples, and interview questions highlighted
- **Wikilinks** — every topic links to related notes (graph view shows connections)
- **YAML frontmatter** — compatible with Dataview plugin for querying
- **Interview Q&A** — common follow-up questions with answers in each note

## Recommended Study Order

1. Read [[Interview Framework]] first (understand interview structure)
2. Study [[Estimation Cheat Sheet]] (needed in every interview)
3. Work through **Fundamentals** (01-05)
4. Work through **Core Components** (06-10)
5. Study **Databases** (11-15)
6. Study **Distributed Systems** (16-20)
7. Practice **System Designs** (SD01-SD12) out loud (time yourself: 45 min each)
8. Review **Common Questions & Answers**

---

*Based on Gaurav Sen's System Design playlist. Structured for 5+ years experience preparing for FAANG-level interviews.*
