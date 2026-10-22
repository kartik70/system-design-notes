---
tags: [system-design, interview, framework, faang]
aliases: [Interview Framework, SD Interview, How to Approach]
created: 2026-10-22
difficulty: ⭐
type: framework
---

# Interview Framework — How to Ace the System Design Interview

> [!important] Time Budget (45-60 min interview)
> 5 min — Requirements clarification  
> 5 min — Capacity estimation  
> 5 min — API design  
> 10 min — High-level design  
> 15 min — Deep dive into 2-3 components  
> 5 min — Trade-offs and scaling  
> 5 min — Buffer / follow-up questions

---

## Step 1: Requirements Clarification (5 min) ✅

**NEVER start designing without this.** Interviewers change requirements mid-interview if you skip this.

### What to Ask

**Scope:**
- Which features are in scope? (Start with MVP)
- Which users/use cases should I prioritize?

**Scale:**
- How many users? DAU? Concurrent users?
- Read/write ratio?
- Data size? Growth rate?

**Constraints:**
- What's the latency requirement?
- What's the availability requirement (SLA)?
- Any specific technology constraints?

**Non-obvious:**
- Is real-time required, or eventual consistency OK?
- Multi-region or single region?
- Mobile-only or also web?

> [!example] Good vs Bad Requirements Gathering
> ❌ Bad: "I'll design a URL shortener" (immediately starts drawing)  
> ✅ Good: "Before I start, let me clarify scope. Should I support analytics? Custom aliases? Expiration? Approximately how many URLs per day? What's the read:write ratio?"

---

## Step 2: Capacity Estimation (5 min) ✅

Don't skip this — it tells you whether you need 1 server or 1000 servers.

### Template

```
Step 1: QPS (Queries Per Second)
  Daily Active Users: [DAU]
  Actions per user per day: [N]
  Write QPS = DAU × N / 86,400
  Read QPS = Write QPS × [read:write ratio]

Step 2: Storage
  Data per record: [bytes]
  New records/day: [write QPS × 86,400]
  Retention: [years]
  Total = data_per_record × records/day × days × retention_years

Step 3: Bandwidth
  Write bandwidth = Write QPS × avg_record_size (MB/s)
  Read bandwidth = Read QPS × avg_record_size (MB/s)
  
Step 4: Cache
  Working set (top 20% of data) = Total_storage × 0.20
```

### Key Conversions to Memorize
```
1M/day ≈ 12/sec
1B/day ≈ 12K/sec
1B/month ≈ 400/sec
```

---

## Step 3: API Design (5 min) ✅

Define the key API endpoints before drawing architecture.

```
For each major feature:
  Method: GET/POST/PUT/DELETE
  Endpoint: /api/v1/resource
  Request body: key fields
  Response: key fields + status code
  
Don't over-specify — focus on 2-3 core endpoints
Mention pagination, auth, rate limiting at endpoint level
```

> [!tip] Skip CRUD, Focus on Core
> Don't define 20 endpoints. Define the 3-4 that are architecturally interesting (the write path, the read path, anything with special requirements).

---

## Step 4: High-Level Design (10 min) ✅

Draw the architecture diagram with major components. Start simple, then evolve.

### Template Architecture

```mermaid
graph LR
    Client --> CDN[CDN\n(static assets)]
    Client --> LB[Load Balancer]
    LB --> API[Stateless\nAPI Servers]
    API --> Cache[(Cache\nRedis)]
    API --> DB[(Primary DB)]
    DB --> Replica[(Read Replicas)]
    API --> MQ[Message Queue\nKafka]
    MQ --> Workers[Async Workers]
```

### Drawing Tips
- Start with client on left, storage on right, flow left to right
- Name every component (not just "servers")
- Show data flow with arrows
- Distinguish sync calls (solid) from async (dotted)

---

## Step 5: Deep Dive (15 min) ✅

Pick 2-3 components to go deep on. Usually the interviewer will guide you.

**Good deep-dive topics:**
- Database schema design
- Caching strategy
- Feed generation algorithm
- Real-time communication (WebSockets)
- Specific algorithm (consistent hashing, Trie, etc.)

**Deep dive structure:**
1. State the problem clearly
2. Propose a naive solution
3. Identify its weakness
4. Propose the better solution
5. Discuss trade-offs

> [!example] Good Deep Dive Pattern
> "For the timeline cache, the simplest approach is to fetch all posts from all followees on every read — but with 500 followees this is 500 DB queries. Instead, I'd pre-compute the feed using fan-out on write..."

---

## Step 6: Scale & Trade-offs (5 min) ✅

Address bottlenecks and show you understand trade-offs.

**Common bottlenecks to address:**
- Single database → add replicas, then sharding
- Single cache → Redis cluster with consistent hashing
- CPU-bound processing → more servers, auto-scaling
- I/O-bound → async processing via message queue
- Network bandwidth → CDN for static/media content

**Trade-offs framework:**
```
Every design decision has a trade-off. State both sides:

"I chose fan-out on write because it gives fast reads,
 at the cost of higher write amplification. This is
 acceptable because our system is 95% reads."

"I chose eventual consistency because it allows higher
 availability and lower latency. The business accepts
 that users may see slightly stale feeds (< 10 seconds)."
```

---

## What FAANG Interviewers Look For

### Green Flags ✅
- Asks clarifying questions before starting
- States assumptions explicitly ("I'm assuming single-region for now")
- Evolves design from simple to complex (doesn't start with distributed everything)
- Justifies choices ("I chose Redis because...")
- Knows trade-offs ("This approach has the downside of...")
- Brings up failure cases proactively
- Mentions monitoring/observability
- Knows specific technologies and when to use them (not just "a database")

### Red Flags ❌
- Jumps to implementation without requirements
- Uses buzzwords without understanding (blockchain, ML, microservices for everything)
- Can't explain trade-offs
- Designs perfect system without acknowledging complexity/cost
- Doesn't know latency numbers (can't reason about performance)
- Doesn't handle failure cases
- Gets defensive when interviewer challenges design

---

## Common Follow-up Questions

```
"What happens if [component] goes down?"
→ Discuss failover, replication, circuit breaker

"How would you scale this to 10x?"
→ Discuss adding replicas, sharding, caching

"What if the requirements changed to include [new feature]?"
→ Show how your architecture accommodates it

"Why did you choose X over Y?"
→ State trade-offs explicitly

"What would you monitor in production?"
→ Mention: latency P99, error rate, throughput, queue depth, cache hit rate

"What would you do first in a production incident?"
→ Runbook approach: check dashboards, identify bottleneck, apply mitigation
```

---

## Interview Checklist

- [ ] Ask requirements before starting
- [ ] Estimate scale (QPS, storage) before designing
- [ ] Define 2-3 key API endpoints
- [ ] Draw simple architecture first, then evolve
- [ ] Go deep on 2-3 interesting components
- [ ] State all trade-offs explicitly
- [ ] Address failure modes
- [ ] Discuss monitoring and alerting

---

## Related Notes

- [[Estimation Cheat Sheet]] — quick reference during estimation
- [[Common Questions & Answers]] — specific Q&A
- [[🗺️ System Design MOC]] — all topics
