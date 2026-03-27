# Technical and Operational Features

## Written in ANSI C

### What is ANSI C?
ANSI C is the standardized C language specification defined by the American National Standards Institute (ANSI).

### Why does it matter?
Because Redis is implemented in standard C, it remains lightweight with minimal dependencies.
It runs reliably across many operating systems and is highly optimized to utilize hardware efficiently.

---

## Lua Scripting and Atomicity

### What is Lua?
Lua is a lightweight embedded scripting language built into Redis.
It allows you to execute logic directly on the Redis server.

### Why is it powerful?
When you send multiple operations as one Lua script, Redis executes that script as a single uninterrupted unit.
Because Redis processes commands in a single-threaded event loop, script execution is naturally atomic (no explicit lock needed).

This gives you:
- Correct atomic execution for complex business logic
- Fewer network round trips (multiple operations in one request), reducing RTT overhead

Keep scripts short and bounded — a long-running script blocks the event loop and can hurt latency for all other clients.

---

## Replication and Cluster

### Replication
Replica servers ensure service continues even if one node fails (**high availability**).

### Cluster
Shard data across multiple servers to scale memory and throughput horizontally (**scalability**).

---

## License — Recent Changes

Redis used a permissive BSD license for a long time, but licensing changed in recent versions (RSALv2/SSPL).
For most internal application teams, day-to-day usage is often unaffected, but verify with your legal/compliance team. Cloud service providers face additional restrictions, so it is worth understanding the policy change at a high level.