# JSON, Search & Probabilistic Structures

For years these features shipped as separate **modules** (RedisJSON, RediSearch, RedisBloom, RedisTimeSeries), bundled together as **Redis Stack**. Starting with **Redis 8**, they are part of the standard Redis distribution, so `JSON.SET`, `FT.SEARCH`, and `BF.ADD` work out of the box.

> **Compatibility note:** On Redis 7.x or many managed services, these commands exist only if the corresponding module is installed. Valkey has its own modules (valkey-json, valkey-bloom, valkey-search, shipped together as valkey-bundle) with similar commands, but compatibility is not guaranteed. Check with `MODULE LIST` or `COMMAND INFO JSON.SET` before you depend on them.

## JSON

Stores a real JSON document under a key and lets you read or update **parts** of it by JSONPath, instead of re-serializing a whole String.

```bash
127.0.0.1:6379> JSON.SET product:56 $ '{"name":"Wireless Keyboard","price":49.99,"tags":["usb","bt"],"stock":10}'
OK

127.0.0.1:6379> JSON.GET product:56 $.price
"[49.99]"

# Atomic partial updates
127.0.0.1:6379> JSON.NUMINCRBY product:56 $.stock -1
"[9]"
127.0.0.1:6379> JSON.ARRAPPEND product:56 $.tags '"wireless"'
1) (integer) 3
```

**When to use it over a Hash:** nested objects and arrays. For flat records, a Hash is simpler and usually smaller.

## Search (Query Engine)

Secondary indexes over Hashes or JSON documents: full-text search, numeric/tag filters, sorting, and aggregations.

```bash
# Index every JSON key starting with "product:"
127.0.0.1:6379> FT.CREATE idx:products ON JSON PREFIX 1 product: SCHEMA $.name AS name TEXT $.price AS price NUMERIC $.tags[*] AS tags TAG
OK

# Full-text + numeric range + tag filter
127.0.0.1:6379> FT.SEARCH idx:products "keyboard @price:[0 100] @tags:{bt}" RETURN 2 name price
1) (integer) 1
2) "product:56"
3) 1) "name"
   2) "Wireless Keyboard"
   3) "price"
   4) "49.99"
```

The index is updated automatically when matching keys are written—no dual writes in your application.

## Probabilistic structures

| Structure | Answers | Commands |
|---|---|---|
| **Bloom filter** | "Have I *possibly* seen X?" (no false negatives) | `BF.RESERVE`, `BF.ADD`, `BF.EXISTS` |
| **Cuckoo filter** | Same, but supports deletion | `CF.ADD`, `CF.EXISTS`, `CF.DEL` |
| **Count-Min Sketch** | "Roughly how often did X occur?" | `CMS.INITBYDIM`, `CMS.INCRBY`, `CMS.QUERY` |
| **Top-K** | "What are the heaviest hitters?" | `TOPK.RESERVE`, `TOPK.ADD`, `TOPK.LIST` |
| **t-digest** | Percentiles (p50/p99) over a stream of values | `TDIGEST.CREATE`, `TDIGEST.ADD`, `TDIGEST.QUANTILE` |

```bash
# Bloom filter: 1% false-positive rate, sized for 1M items
127.0.0.1:6379> BF.RESERVE signup:emails 0.01 1000000
OK
127.0.0.1:6379> BF.ADD signup:emails "alice@corp.io"
(integer) 1
127.0.0.1:6379> BF.EXISTS signup:emails "alice@corp.io"
(integer) 1
127.0.0.1:6379> BF.EXISTS signup:emails "bob@corp.io"
(integer) 0
```

## Practical patterns

- **Cache penetration guard:** put all valid IDs in a Bloom filter; requests for IDs that are definitely absent never reach the DB (see Chapter 6).
- **Product catalog search:** JSON documents + `FT.CREATE` give filtering and full-text search without running a separate search cluster for small/medium catalogs.
- **Trending items:** `TOPK.ADD` on every view, `TOPK.LIST` for the dashboard.

## Pitfalls

- **Portability.** Code using `JSON.*` / `FT.*` / `BF.*` is tied to Redis 8+ or module-enabled deployments. Verify your managed provider supports them.
- **Index memory.** Search indexes live in RAM next to your data; a TEXT index on large documents can cost as much as the documents.
- **Bloom filters can't shrink or delete.** Size them upfront (`BF.RESERVE`); use a Cuckoo filter if you need deletes.
- **Cluster.** `FT.SEARCH` across a cluster needs a distributed setup supported by your deployment; check before you shard.

## Things to think about

1. Redis 8 folded the former Redis Stack modules into core Redis.
2. JSON for nested documents, Search for secondary indexes, probabilistic structures for "approximately" questions at tiny memory cost.
3. Check availability per environment—older Redis needs the modules, and Valkey offers its own equivalents (valkey-json, valkey-bloom, valkey-search) whose syntax and compatibility may differ.
