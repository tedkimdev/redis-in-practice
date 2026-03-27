# Redis Characteristics

## Why is Redis single-threaded (for command execution)?

Redis processes commands on a single-threaded execution path, meaning commands are handled one at a time.

"Wouldn't more threads always be faster?"
Not necessarily. A single-threaded execution model can provide real performance benefits in this workload.

### Why single-threaded can be fast

1. **Lower command-path context-switch overhead**
   The CPU does not constantly switch between worker threads for command logic.

2. **Less lock contention**
   Multiple threads are not competing to mutate the same in-memory structures at the same time, so command-path concurrency control is simpler.

3. **Often bottlenecked by memory/network, not raw CPU**
   For many Redis use cases, latency is constrained more by memory access and network I/O than by pure CPU compute.

---

## Technology behind Redis speed: asynchronous I/O multiplexing

Single-threaded command execution does **not** mean Redis "just waits."
Redis uses asynchronous I/O multiplexing so one process can monitor many client sockets and quickly process ready requests.

In practice, this allows one Redis node to handle large numbers of concurrent connections efficiently — though actual throughput depends on workload, command mix, and hardware.

The trade-off is that any long-running command (e.g., a heavy Lua script or `KEYS *`) blocks the event loop and delays all other clients until it finishes.

> **Note:** Since Redis 6, additional I/O threads can handle network read/write to improve throughput, but command execution remains single-threaded for deterministic behavior.

---

## Persistence

### RDB (Redis Database) — snapshot-based

> **Note:** In this context, **RDB** means **Redis Database snapshot format**, not **Relational Database** (RDBMS).

At configured points in time, Redis writes an in-memory snapshot to a `.rdb` file (for example, every hour based on save rules).

**Pros:**
- Compact file format
- Fast restore time (binary snapshot)

**Cons:**
- Data written after the last snapshot can be lost if a crash occurs before the next snapshot

### AOF (Append Only File) — log-based

Redis appends write commands (e.g., `SET`, `DEL`) to an `.aof` log.

**Pros:**
- Better durability characteristics (depending on fsync policy)

**Cons:**
- Log files can grow large
- Replay-based recovery is generally slower than loading RDB snapshots

### Practical tip

Many production systems use **both**:
- AOF for stronger durability
- Periodic RDB snapshots for faster recovery and operational convenience

---

## Key takeaways

1. **In-Memory** — Redis stores working data in RAM, delivering very low-latency responses (often sub-millisecond).
2. **Single-threaded execution path** — Sequential command processing reduces contention, while asynchronous I/O multiplexing handles many concurrent connections.
3. **Persistence** — Choose RDB, AOF, or both based on how much data loss and recovery time your system can tolerate.