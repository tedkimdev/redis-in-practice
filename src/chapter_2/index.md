# Chapter 2: Redis Basic Concepts

## Intro

This chapter covers foundational ideas: what Redis is, how it stores data in memory, core data types, and the mental model you need before diving into patterns and production use.

## Redis Key Points

**Performance**
- **In-memory architecture** — Uses RAM to deliver extremely low-latency access.
- **Single-threaded core (I/O multiplexing)** — Reduces contention in the command execution path by processing commands sequentially.
- **ANSI C implementation** — Lightweight, fast, and highly portable across operating systems.

**Function**
- **Rich data types** — Supports structures from Strings to Sorted Sets for many workload patterns.
- **Data persistence** — Supports disk durability with RDB snapshots and AOF logs.
- **Lua scripting** — Runs server-side logic atomically inside Redis.

**Operations / Scale**
- **Replication & Cluster** — Improves availability and supports distributed data storage.
- **Multi-purpose usage** — Works for cache, queue-like patterns, Pub/Sub, and more.
- **License changes** — License moved from BSD to RSALv2/SSPL; understand implications.