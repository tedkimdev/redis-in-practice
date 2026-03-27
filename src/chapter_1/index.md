# Chapter 1: Introduction

## Intro

Welcome to *Redis in Practice*. This chapter introduces why Redis matters, how it fits next to databases and caches, and what you can expect from the rest of the book.

## Why use Redis?

### The backend challenge in the era of exploding data

Modern services face a different set of constraints than the past:

1. **A sudden increase in users:** millions of concurrent connections.
2. **More complex data needs:** not just simple lookups, but computation-heavy workloads like real-time rankings and recommendations.
3. **User experience expectations:** even a 0.1 second delay can be noticeable—and costly.

### What happens if you rely on an RDBMS alone?

As traffic grows, an RDBMS can hit **disk I/O bottlenecks**. CPU usage climbs, queues build up, and end-to-end latency increases rapidly. Simply upgrading the database server (**vertical scaling / scale-up**) is often not cost-effective.

### Redis as a practical solution

This is where Redis comes in. Redis is an **in-memory data structure store**, designed to keep hot data in memory and serve it quickly.

- **Relieve bottlenecks:** cache frequently accessed data to reduce disk reads.
- **Accelerate applications:** return computed results or session data directly from memory with very low latency.

### Disk vs Memory latency

Approximate access latency (order-of-magnitude):

- **L1 cache:** ~0.5 ns (0.3–1 ns)
- **Main memory (RAM):** ~100 ns (80–120 ns)
- **SSD:** ~150 µs (80–200 µs)  
  - Modern high-performance SSDs can be ~40–100 µs
- **HDD:** ~5–10 ms

### Things to think about

1. **Disk is durable, but slow.**
2. **Memory is fast, but expensive and volatile.**
3. **Redis turns RAM speed into application performance.**
4. **Question:** “So should we put *all* data in Redis?”
    <details>
    <summary><strong>Answer</strong></summary>

    **Usually no.**

    Redis is excellent for **caching** and **fast shared state**, but it’s not always the best place to store *all* data because memory is expensive and some workloads need stronger durability guarantees.

    A common production approach is:
    - **RDBMS as the source of truth** (durable, consistent)
    - **Redis as an acceleration layer** (cache hot reads, store sessions, rate limits, queues/streams, distributed coordination)
    </details>

## Data consistency in distributed systems

### Local cache

A local cache keeps data inside the application process—stored in variables like a `HashMap` / `ConcurrentHashMap`.

**Characteristics**
- Faster than Redis because it avoids the network (it’s your server’s local memory).

**Limitations**
- It disappears when the server restarts.
- It can only be accessed by that single server instance.

### The disaster in distributed systems: inconsistency

Modern services don’t run on a single server. As traffic grows, we scale out to multiple instances. If each instance uses its own local cache, you can end up with **different cached values on each server**.

When users are routed to different servers, they may see data “randomly” change between requests. This is **data inconsistency**.

### Global cache

**Solution: Redis as a global cache.**  
Redis acts as a shared, central in-memory store that all application servers read from and write to.

### Things to think about

1. When you’re developing alone—or running a single server—local caching is convenient. In a multi-server (distributed) environment, a global cache like Redis becomes essential.
2. Redis is not just a “fast database.” It’s a shared in-memory system that helps multiple servers stay consistent by using a single cache source.

## Hybrid approach with an RDBMS

**RDBMS (MySQL, PostgreSQL, etc.)**: reliability first. It is the **system of record**—the authoritative source of truth.  
**Redis**: speed first. It is a **temporary store** for hot data and a place to perform fast operations.

### Most common strategy: Cache-Aside (Look-aside)

The application checks the cache (Redis) first. If the value is missing, it falls back to the database.

**Read flow**
1. The application checks Redis for the key.
2. **Cache hit:** if Redis has the data, return it immediately (fastest path).
3. **Cache miss:** if Redis does not have the data, read it from the RDBMS (and typically populate Redis afterward).

### Write strategies: Write-Through vs Write-Back

- **Write-Through:** when data changes, update **both Redis and the database** as part of the write path (aims to keep cache fresh).
- **Write-Back (Write-Behind):** write to Redis first, then flush/batch writes to the database later (used to maximize write throughput, e.g., logs, engagement counters, view counters).

### Why hybrid?

1. **Risk of data loss:** Redis is memory-based, so data can be lost during failures. Important data should remain in the database.
2. **Cost efficiency:** instead of storing everything in expensive RAM, keep the hottest ~20% in Redis and the rest on cheaper disk storage (the Pareto principle).

### Things to think about

1. Redis is a "helper" for the RDBMS: it sits in front of the database and absorbs traffic like a shield.
2. If you understand the Cache-Aside pattern well, you’ve already learned a large part of backend performance optimization.
3. Question: "What happens when Redis is full? Do we need to delete old data?"
    <details>
    <summary><strong>Answer</strong></summary>

    **Yes.** Redis provides **eviction policies** and **TTL (time-to-live)** settings to automatically remove old or less-used data when memory is full. We'll cover these mechanisms in detail in a later chapter.
    </details>
