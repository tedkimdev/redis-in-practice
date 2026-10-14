# Redis 7.x/8, the License Change, and Valkey

Many Redis blog posts and courses were written in the 5.0–6.x era. Redis 7.x and 8.0 changed a lot of internals and added features that change how you design systems—and the project's licensing and community also split. This section covers what changed and why it matters.

## Version timeline

| Version | Period | Theme |
|---|---|---|
| **5.x – 6.x** | 2018–2021 | Streams, listpack introduced, module ecosystem grows |
| **7.0 – 7.2** | 2022–2023 | Internal architecture: listpack everywhere, multi-part AOF, Functions, sharded Pub/Sub |
| **7.4** | 2024 | Hash field expiration; first release under the new license |
| **8.0** | GA May 2, 2025 | Multi-model platform: former Redis Stack features built in |

The direction is clear: **from a cache server to a data platform.**

## Memory and persistence (7.0+)

### Ziplist is gone, listpack everywhere

Small Hashes, Lists, and Sorted Sets used to be stored in a compact **ziplist**. Each ziplist entry stores the previous entry's length, so growing one entry could force every following entry to be rewritten—a *cascading update*. **Listpack** (introduced in 5.0) stores each entry's own length instead, so there are no cascades. In **7.0** ziplist was removed entirely. Commands are unchanged; the internals are faster and more predictable.

### Multi-part AOF

Before 7.0, AOF was one ever-growing file, and rewriting it was heavy on disk I/O and latency. **7.0** splits it into a **base** file (RDB or AOF format) plus **incremental** AOF files tracked by a manifest, making rewrites lighter. Persistence is covered in more detail in a later chapter.

## Server-side logic: Redis Functions (7.0+)

`EVAL` scripts live in the application and are re-sent over the network; changing them means redeploying every client. **Functions** register Lua code on the server as named functions—similar to stored procedures—so every service calls the same logic by name. See [Transactions & Scripting](./transactions_and_scripting.md).

## Cluster-friendly messaging: Sharded Pub/Sub (7.0+)

Classic Pub/Sub in Redis Cluster **broadcasts** every message to every node, so traffic grows with cluster size. **Sharded Pub/Sub** hashes the channel name to a slot, just like a key, and delivers only within that shard.

```bash
127.0.0.1:6379> SSUBSCRIBE event:{order}:completed
127.0.0.1:6379> SPUBLISH event:{order}:completed "order_id=123"
```

With a hash tag (`{order}`), `event:{order}:created`, `event:{order}:completed`, and `event:{order}:cancelled` all land on the same shard.

## Atomic convenience commands

Every round trip (RTT) costs time, and every gap between two commands is a chance for another client to slip in. Recent versions keep merging common multi-command sequences into **single atomic commands**.

### Hash field expiration (7.4+)

Until 7.4, TTL applied only to whole keys. Now individual Hash fields can expire:

```bash
127.0.0.1:6379> HSET user:session:42 token "abc" otp "482901" last_access "1760470000"
(integer) 3
127.0.0.1:6379> HEXPIRE user:session:42 180 FIELDS 1 otp
1) (integer) 1
127.0.0.1:6379> HTTL user:session:42 FIELDS 2 token otp
1) (integer) -1
2) (integer) 180
```

Redis 8.0 added Hash commands that combine read/write with deletion or expiry:

| Command | What it does | Replaces | Typical use |
|---|---|---|---|
| `HGETDEL` | Get fields and delete them | `HGET` + `HDEL` | OTPs, one-time tokens |
| `HGETEX` | Get fields and set/refresh their TTL | `HGET` + `HEXPIRE` | Sliding sessions |
| `HSETEX` | Set fields with a TTL | `HSET` + `HEXPIRE` | Short-lived partial caches |

### Other handy additions

| Command | Since | What it does | Replaces |
|---|---|---|---|
| `GETDEL` | 6.2 | Get a String and delete it | `GET` + `DEL` |
| `GETEX` | 6.2 | Get a String and set its TTL | `GET` + `EXPIRE` |
| `SMISMEMBER` | 6.2 | Check many Set members at once | repeated `SISMEMBER` |
| `LMPOP` / `BLMPOP` | 7.0 | Pop from the first non-empty of several Lists | repeated `LPOP` / `BLPOP` |
| `ZMPOP` / `BZMPOP` | 7.0 | Same for Sorted Sets | repeated `ZPOPMIN` |
| `WAITAOF` | 7.2 | Block until writes are fsynced to AOF on the primary and/or replicas | — |

> **Pipelining vs atomic commands:** a pipeline also cuts RTT, but other clients' commands can still run between the pipelined commands. An atomic command gives you both fewer round trips **and** atomicity.

## Redis 8: built-in modules

1. **Module era** — JSON, Search, TimeSeries, and Bloom were separate modules you built and loaded yourself.
2. **Redis Stack (2022–)** — the popular modules bundled into one package.
3. **Redis 8** — JSON, the Query Engine (search and vectors), time series, and five probabilistic structures ship in the standard distribution, plus the new **Vector Set** type (beta). The product is now called **Redis Open Source**, and Redis Stack is effectively merged into it.

Practically: install plain Redis 8 and you can store JSON documents and search them right away. See [JSON, Search & Probabilistic Structures](./json_search_probabilistic.md) and [Vector Search](./vector_search.md).

## The license change

| Version | License |
|---|---|
| ≤ 7.2.x | BSD-3-Clause (7.2 is the last BSD release line; 7.2.x patch releases stay BSD) |
| 7.4.x – 7.8.x | Dual RSALv2 / SSPLv1 (change announced **March 2024**) |
| ≥ 8.0 | Your choice of RSALv2, SSPLv1, or **AGPLv3** (added with 8.0 GA, May 2025) |

RSALv2 and SSPLv1 are "source available," not OSI-approved; the change targeted cloud providers selling Redis as a managed service. AGPLv3 *is* OSI-approved open source, but it is copyleft: if you modify Redis and offer it over a network, you must publish your changes.

**What it means for you:** using Redis inside your own application is generally unaffected. If you redistribute or host Redis for others, have your legal team check which license fits.

## Valkey

In March 2024, right after the license change, former Redis contributors—backed by AWS, Google Cloud, Oracle, and others—forked **Redis 7.2.4** as **Valkey** under the **Linux Foundation**, keeping BSD-3-Clause.

- Valkey is protocol- and command-compatible with Redis OSS 7.2, so existing clients work unchanged. (Data files from Redis 7.4+ are *not* compatible with Valkey.)
- Valkey 8.x focused on performance (improved multi-threaded I/O) and memory efficiency. **Valkey 9.0** (October 2025) added Hash field expiration (`HEXPIRE`, `HGETEX`, `HSETEX`, …).
- Module equivalents exist: **valkey-json**, **valkey-bloom**, and **valkey-search** (including vector search), shipped together as **valkey-bundle**. Command syntax and compatibility with the Redis versions may differ, so test before you switch.
- Several managed services offer Valkey (e.g. AWS ElastiCache / MemoryDB, Google Cloud Memorystore), often cheaper than their Redis equivalents.

## Other trends worth knowing

- **Compatible alternatives:** Dragonfly and Microsoft Garnet implement the Redis protocol with different threading/storage models. KeyDB, once a popular multithreaded fork, is now largely inactive. Compatibility is usually high for core commands and weaker for scripting, modules, and Cluster edge cases.
- **Managed / serverless Redis:** most teams pick a managed offering and inherit its engine, version, and feature limits.
- **Redis as an AI building block:** vector search and semantic caching are now a major reason teams adopt Redis.

## Pitfalls

- **"Redis" in your cloud console may not be Redis 8.** Check the actual engine and version before using `JSON.*`, `FT.*`, `HGETDEL`, or `VADD`.
- **Drifting forks.** Redis and Valkey share a 7.2 base and diverge after it. Stick to the common subset if you need portability.
- **Licensing assumptions.** Know which license applies to the version you ship.

## Things to think about

1. Internals: listpack and multi-part AOF make modern Redis leaner and more stable than older versions.
2. Features: put complex logic in Functions, and use Sharded Pub/Sub for large-scale notifications in Cluster.
3. Atomicity and RTT: Hash field TTLs and atomic convenience commands (`HGETDEL`, `GETEX`, …) remove whole classes of race conditions.
4. Redis 8 is a multi-model platform; pick your engine (Redis, Valkey, alternatives) deliberately and code to the feature set it actually supports.
