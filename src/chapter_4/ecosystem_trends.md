# Redis 7.x/8, the License Change, and Valkey

Redis the *software* changed a lot between 2022 and 2025—not only in features, but in who maintains it and under what license. If you choose or operate Redis today, you need to know this history.

## Version highlights

| Version | Notable changes |
|---|---|
| **7.0** (2022) | Redis Functions (`FUNCTION LOAD`, `FCALL`) as a managed successor to ad-hoc `EVAL` scripts; sharded Pub/Sub; multi-part AOF; ACL v2 improvements |
| **7.2** (2023) | Last release under the BSD license; performance and listpack improvements |
| **7.4** (2024) | First release under the new dual license (RSALv2 / SSPLv1); per-field expiration for Hashes (`HEXPIRE`, `HTTL`, …) |
| **8.0** (2025) | JSON, Query Engine (search & vectors), time series, and probabilistic structures folded into core; new Vector Set type (beta); AGPLv3 added as a license option; many performance improvements |

### Hash field expiration (7.4+)

Before 7.4, TTL only applied to whole keys. Now individual Hash fields can expire:

```bash
127.0.0.1:6379> HSET session:42 token "abc" csrf "xyz"
(integer) 2
127.0.0.1:6379> HEXPIRE session:42 300 FIELDS 1 csrf
1) (integer) 1
127.0.0.1:6379> HTTL session:42 FIELDS 2 token csrf
1) (integer) -1
2) (integer) 300
```

## The license change

- Until 7.2, Redis was licensed under **BSD-3-Clause**—anyone, including cloud providers, could host it as a service.
- In **March 2024**, Redis Ltd. moved new versions (7.4+) to a dual **RSALv2 / SSPLv1** license. Both are "source available" rather than OSI-approved open source; the practical effect is restricting offering Redis as a competing managed service.
- In **May 2025**, with Redis 8, Redis added **AGPLv3** as a third option, making it OSI-approved open source again (under a copyleft license).

**What it means for you:** using Redis inside your own application or company is generally unaffected. The restrictions target companies that sell Redis itself as a service. As always, have your legal team check your specific case.

## Valkey

Right after the 2024 change, former Redis contributors (with backing from AWS, Google Cloud, Oracle, and others) forked the last BSD version, 7.2.4, as **Valkey**, under the **Linux Foundation**, keeping the BSD license.

- Valkey is wire- and command-compatible with Redis 7.2, so existing clients (`go-redis`, `redis-rs`, `redis-py`, …) work unchanged.
- Valkey 8.x focused on performance (e.g. improved multi-threaded I/O) and memory efficiency.
- Several managed services now offer Valkey (e.g. AWS ElastiCache / MemoryDB for Valkey, Google Cloud Memorystore for Valkey), often at lower prices than their Redis equivalents.
- Features added to Redis after the fork (Hash field expiration in 7.4, the Redis 8 built-in modules, Vector Sets) are **not automatically** in Valkey; Valkey has its own roadmap and module ecosystem.

## Other trends worth knowing

- **Compatible alternatives:** Dragonfly, KeyDB, and Garnet implement the Redis protocol with different threading/storage models. Compatibility is usually high for core commands and weaker for scripting, modules, and Cluster edge cases.
- **Managed / serverless Redis:** most teams no longer run Redis on bare VMs; they pick a managed offering and inherit its version and feature limits.
- **Redis as an AI building block:** vector search and semantic caching (see [Vector Search](./vector_search.md)) are now a major reason teams adopt Redis.

## Pitfalls

- **"Redis" in your cloud console may not be Redis 8.** Check the actual engine and version before using `JSON.*`, `FT.*`, `HEXPIRE`, or `VADD`.
- **Drifting forks.** Redis and Valkey share a 7.2 base but diverge after it. Stick to the common subset if you want to stay portable.
- **Licensing assumptions.** "It's open source, so anything goes" is no longer a safe default—know which license applies to the version you ship.

## Things to think about

1. 7.2 is the last BSD Redis and the base of Valkey.
2. Redis 8 brings former Stack modules into core and adds AGPLv3 as an option.
3. Pick your engine (Redis, Valkey, compatible alternatives) deliberately, and code to the feature set it actually supports.
