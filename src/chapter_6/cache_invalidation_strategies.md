# Practical Cache Invalidation Strategies

There's a famous joke: the two hardest problems in computer science are cache invalidation, naming things, and off-by-one errors. This chapter is about the first one.

We'll go from simple TTL to CDC pipelines and multi-layer hybrid architectures. If you're in a hurry — start with Cache-Aside + TTL with Jitter. That alone covers most real-world cases. Come back here when it's not enough.

---

## Why Should You Care About Stale Data?

Caching makes things fast. But stale data causes real problems — not just slow pages, but actual business damage:

| Domain | What happens | The cost |
|---|---|---|
| **E-commerce** | Stock is 0 but cache says 10 | Over-selling, refund headaches |
| **Security** | Banned user still has a valid session | They keep doing damage |
| **Fintech** | Deposit landed but balance didn't update | User panics, tries to transfer again |

The point: stale data isn't a minor inconvenience. In some domains it's a correctness bug with real money attached.

---

## Cache Storms (Thundering Herd)

Here's a scenario that catches people off guard. You have a popular cache key. It expires. Suddenly thousands of requests slam the database at the same time.

What it looks like:
- Latency spikes right at the TTL boundary
- DB CPU hits 100%
- Connection pool runs dry
- Everything downstream starts failing

This is called the **Thundering Herd** problem, and every strategy below needs to account for it.

---

## The Five Strategies

Here's the roadmap. Each one builds on the last:

1. **Passive (TTL)** — set it and forget it
2. **Active (Write-Driven)** — app deletes/updates cache on write
3. **Event-Driven** — publish change events, subscribers invalidate
4. **CDC** — capture changes from the DB log directly
5. **Hybrid** — combine multi-layer caching with CDC

Pick the simplest one that meets your consistency needs.

---

## 1. Passive: TTL-Based

### Static TTL

The simplest approach. Give each data type a fixed expiration:

```
SET product:123 "{...}" EX 300    # 5 min
SET config:flags "{...}" EX 3600  # 1 hour
```

This works surprisingly well for data that doesn't need to be fresh — feature flags, config values, reference data.

The catch: TTL alone won't save you from read-after-write issues. If a user updates their profile, they might see the old version until the key expires. For user-facing writes, you'll want to pair TTL with explicit cache deletion (see Cache-Aside below).

#### Jitter

If you set the same TTL on thousands of keys, they all expire at once — hello, Thundering Herd. The fix is dead simple: add some randomness.

```go
baseTTL := 300 * time.Second
jitter := time.Duration(rand.Intn(60)-30) * time.Second
rdb.Set(ctx, key, value, baseTTL+jitter)
```

That's it. +/- 30 seconds of noise spreads the expirations out.

### Adaptive TTL

Some data changes every second, some hasn't changed in months. Why give them the same TTL?

The idea: track when each key was last modified. If it was just updated, it's probably hot — give it a short TTL. If it hasn't changed in a while, give it a long one.

#### Lua Script Implementation

We store the data and a `last_mod` timestamp together in a Redis Hash, and let a Lua script figure out the right TTL:

**Key:** `user:profile:123` → Hash with fields `data` and `last_mod`

The logic:
1. Read the previous `last_mod`
2. Calculate the gap (how long since the last change)
3. Small gap = hot data = short TTL. Large gap = cold data = long TTL
4. Set the new data + TTL atomically

```go
var adaptiveTTLScript = redis.NewScript(`
    local key = KEYS[1]
    local data = ARGV[1]
    local now = tonumber(ARGV[2])

    local last_mod = tonumber(redis.call('HGET', key, 'last_mod')) or 0
    local gap = now - last_mod

    redis.call('HSET', key, 'data', data, 'last_mod', now)

    local ttl
    if gap < 10 then
        ttl = 10       -- very hot: 10s
    elseif gap < 60 then
        ttl = 60       -- warm: 1 min
    elseif gap < 300 then
        ttl = 300      -- cool: 5 min
    else
        ttl = 1800     -- cold: 30 min
    end

    redis.call('EXPIRE', key, ttl)
    return ttl
`)

func setWithAdaptiveTTL(ctx context.Context, rdb *redis.Client, key, data string) (int64, error) {
    return adaptiveTTLScript.Run(ctx, rdb, []string{key}, data, time.Now().Unix()).Int64()
}
```

Why Lua? The read-then-write on `last_mod` needs to be atomic. Two concurrent writes without Lua could read the same old timestamp and both compute the wrong TTL.

To read it back, just grab the `data` field:

```go
data, err := rdb.HGet(ctx, key, "data").Result()
```

> If you need to differentiate "updated once 5s ago" from "updated 500 times in 5s" (write volume vs recency), you can add a counter. But the gap-based approach is simpler and handles most cases.

---

## 2. Active: Write-Driven

### Cache-Aside (Lazy Loading)

This is the bread and butter of caching. You've probably already used it:

- **Read:** check cache → miss → query DB → store in cache
- **Write:** update DB → delete the cache key

```go
// Read
val, err := rdb.Get(ctx, key).Result()
if err == redis.Nil {
    val = db.Query(ctx, id)
    rdb.Set(ctx, key, val, ttl)
}

// Write
db.Update(ctx, id, newVal)
rdb.Del(ctx, key)  // delete, don't update
```

**Why delete instead of SET?** If two writes happen concurrently and you SET both times, the cache might end up with the older value. DELETE is safe — the next reader will always fetch fresh data from DB.

For popular keys, you can add a lock so only one goroutine repopulates on a miss (`SET key:lock NX EX 5`). This prevents the Thundering Herd on that specific key.

### Write-Through

Update both DB and cache on every write. In practice, most teams write to the **DB first**, then the cache — so if the cache write fails, you don't need to roll back the DB.

```
Client -> DB (write) -> Cache (write) -> ACK
```

The upside: cache is always fresh right after a write. The downside: every write is slower (two round-trips), and you're caching data that might never be read.

If the cache write fails after the DB commit, you're stale until the next write or TTL. So even here, a fallback TTL is a good idea.

Good for: financial balances, game inventories — anything where "slightly stale" is not acceptable.

### Write-Behind (Write-Back)

The opposite of Write-Through. Write to cache first, respond to the client immediately, then flush to DB asynchronously.

```
Client -> Cache (write) -> ACK
                |
                +---> (async) DB (write)
```

This is fast — really fast. You can batch multiple writes into one DB call. But if Redis dies before flushing, those writes are gone.

You **need** a durability mechanism here. Redis Streams work well as a WAL:

```go
pipe := rdb.Pipeline()
pipe.Set(ctx, key, value, 0)
pipe.XAdd(ctx, &redis.XAddArgs{
    Stream: "write_behind:wal",
    Values: map[string]interface{}{"key": key, "value": value},
})
pipe.Exec(ctx)
```

Good for: like counts, view counters, analytics — high-frequency writes where losing a few on crash is tolerable.

---

## 3. Event-Driven

In a microservice world, one service updates data and others need to know about it:

```
Order Service --[order.updated]--> Kafka ---> Inventory Service (invalidate cache)
                                         ---> Search Service (invalidate cache)
```

Sounds clean. But there are three problems you'll run into:

**Lost events.** If a message is dropped, the cache stays stale forever. Always combine with a TTL as a safety net:

```go
rdb.Set(ctx, key, value, 30*time.Minute) // TTL = insurance policy
```

**Duplicate events.** At-least-once delivery means you'll get duplicates. Luckily, `DEL` is naturally idempotent — deleting a key that doesn't exist is a no-op.

**Out-of-order events.** An older event arriving after a newer one can overwrite fresh data. The fix: attach a version to each event and reject stale ones with a Lua script:

```go
func handleEvent(ctx context.Context, event CacheEvent) error {
    script := redis.NewScript(`
        local cur = redis.call('HGET', KEYS[1], 'version')
        if cur and tonumber(cur) >= tonumber(ARGV[1]) then
            return 0  -- stale, skip
        end
        if ARGV[2] == '__TOMBSTONE__' then
            redis.call('DEL', KEYS[1])
        else
            redis.call('HSET', KEYS[1], 'version', ARGV[1], 'data', ARGV[2])
        end
        return 1
    `)
    data := event.Data
    if event.IsDelete {
        data = "__TOMBSTONE__"
    }
    return script.Run(ctx, rdb, []string{event.Key}, event.Version, data).Err()
}
```

### The big limitation

This whole approach depends on every write path publishing an event. If someone runs a migration, fixes data manually, or another team's service writes to the same table — no event, stale cache. That's where CDC comes in.

---

## 4. CDC (Change Data Capture)

Instead of trusting every service to publish events, read changes directly from the database's transaction log. The DB itself becomes the source of events.

```
PostgreSQL (WAL) --> Debezium --> Kafka --> Consumer --> Redis DEL
```

### Why this beats application-level events

| | App Events | CDC |
|---|---|---|
| Forgot to publish? | Totally possible | Can't happen — DB log captures everything |
| Migration scripts | Missed | Captured |
| Code changes needed | Every write path | Zero |

### The consumer

```go
func consumeCDCEvents(ctx context.Context, reader *kafka.Reader, rdb *redis.Client) {
    for {
        msg, err := reader.ReadMessage(ctx)
        if err != nil {
            log.Printf("read error: %v", err)
            continue
        }

        var event CDCEvent
        json.Unmarshal(msg.Value, &event)

        key, ok := extractCacheKey(event)
        if !ok {
            continue
        }
        rdb.Del(ctx, key)
    }
}

func extractCacheKey(event CDCEvent) (string, bool) {
    switch event.Op {
    case "c", "u": // create/update — use After (the new row)
        if event.After == nil {
            return "", false
        }
        return fmt.Sprintf("%s:%v", event.Source.Table, event.After["id"]), true
    case "d": // delete — use Before (After is nil, the row is gone)
        if event.Before == nil {
            return "", false
        }
        return fmt.Sprintf("%s:%v", event.Source.Table, event.Before["id"]), true
    default:
        return "", false
    }
}
```

One gotcha that's easy to miss: on delete events, the `After` field is nil (the row doesn't exist anymore). You need to pull the ID from `Before`. I've seen this bug in production — the consumer silently skips all deletes and nobody notices until stale data piles up.

### CDC tools by database

| Database | Tool | Source |
|---|---|---|
| PostgreSQL | Debezium, pg_logical | WAL |
| MySQL | Debezium, Maxwell, Canal | Binlog |
| MongoDB | Debezium, Change Streams | Oplog |
| SQL Server | Debezium | CDC tables |

### The trade-off

CDC captures everything and requires zero app code changes. But you're running a Kafka cluster, connectors, and consumers — that's real infrastructure. And there's a propagation delay (100ms-2s) between the DB write and the cache invalidation. For most use cases that's fine, but it's not zero.

---

## 5. Hybrid: Multi-Layer Caching with CDC

This is where it all comes together. In production, the best setups combine multiple strategies.

### Why bother with layers?

Redis is fast (~0.5ms per call). But if you're hitting the same key 10,000 times per second, even 0.5ms adds up. An in-process local cache (L1) gives you nanosecond reads with zero network overhead.

| Layer | Where | Speed | Shared? |
|---|---|---|---|
| **L1** | In-process (`sync.Map`, Caffeine) | Nanoseconds | No — per instance |
| **L2** | Redis | Sub-millisecond | Yes — all instances |
| **DB** | PostgreSQL, etc. | Milliseconds | Source of truth |

### The problem: coherence

If App #1 writes new data, App #2's L1 still has the old version. How does App #2 find out?

With CDC, we get this for free. The CDC consumer invalidates L2 (Redis) and broadcasts a Pub/Sub message. Every app instance listens and clears its L1:

```
DB --WAL--> Debezium --> Kafka --> CDC Consumer --> 1. DEL from Redis (L2)
                                                --> 2. PUBLISH "cache:invalidate"
                                                         |
                                          +--------------+--------------+
                                          |              |              |
                                        App #1         App #2         App #3
                                       L1.Del()       L1.Del()       L1.Del()
```

### Implementation

The CDC consumer side:

```go
func cdcConsumer(ctx context.Context, reader *kafka.Reader, rdb *redis.Client) {
    for {
        msg, err := reader.ReadMessage(ctx)
        if err != nil {
            continue
        }
        var event CDCEvent
        json.Unmarshal(msg.Value, &event)

        key, ok := extractCacheKey(event)
        if !ok {
            continue
        }
        rdb.Del(ctx, key)
        rdb.Publish(ctx, "cache:invalidate", key)
    }
}
```

The application side:

```go
type MultiLayerCache struct {
    l1  *sync.Map
    l2  *redis.Client
    db  *sql.DB
    ttl time.Duration
}

func (c *MultiLayerCache) Get(ctx context.Context, key string) (string, error) {
    // L1 — nanoseconds
    if val, ok := c.l1.Load(key); ok {
        return val.(string), nil
    }
    // L2 — sub-millisecond
    val, err := c.l2.Get(ctx, key).Result()
    if err == nil {
        c.l1.Store(key, val)
        return val, nil
    }
    // DB — milliseconds
    val = c.queryDB(ctx, key)
    c.l2.Set(ctx, key, val, c.ttl)
    c.l1.Store(key, val)
    return val, nil
}

func (c *MultiLayerCache) ListenForInvalidations(ctx context.Context) {
    sub := c.l2.Subscribe(ctx, "cache:invalidate")
    for msg := range sub.Channel() {
        c.l1.Delete(msg.Payload)
    }
}
```

There will always be a small window (a few milliseconds) where L1 is stale after L2 is invalidated — the Pub/Sub message hasn't arrived yet. For user profiles or product metadata, that's fine. For account balances, don't use L1.

### When do you actually need all this?

| Your situation | What to use |
|---|---|
| Single service, moderate traffic | Cache-Aside + TTL is plenty |
| Multiple services writing same data | Add CDC |
| Hot keys getting 10K+ reads/sec | Add L1 |
| Need to capture every change (compliance) | CDC is required |
| All of the above | Full hybrid |

---

## The Safety Net: CDC + TTL

If there's one takeaway from this chapter, it's this: **always set a TTL, even with active invalidation.**

CDC handles the normal case — every DB change invalidates the cache within seconds. But Kafka can go down. Consumers crash. Networks partition. Without a TTL, a lost event means the cache stays stale forever.

```
Happy path:   DB write -> CDC -> cache invalidated     (1-2s)
CDC is down:  DB write -> ...  -> TTL expires           (30 min)
```

```go
// Read path: always set a TTL when populating the cache
val := db.Query(ctx, id)
rdb.Set(ctx, key, val, 30*time.Minute)  // insurance
```

CDC gives you freshness. TTL gives you a guaranteed upper bound on staleness. Together, they're the most robust pattern I know.

---

## Quick Reference

| Strategy | Consistency | Complexity | Best for |
|---|---|---|---|
| **Static TTL** | Eventual | Low | Config, feature flags |
| **Adaptive TTL** | Eventual | Medium | Mixed-frequency data |
| **Cache-Aside** | Eventual | Low | General purpose (start here) |
| **Write-Through** | Strong | Medium | Financial data, inventory |
| **Write-Behind** | Eventual | High | Counters, analytics |
| **Event-Driven** | Near real-time | High | Microservices |
| **CDC** | Near real-time | High | Cross-service, compliance |
| **Hybrid** | Near real-time | Very high | Ultra-hot keys at scale |

---

## Optimization Tips

### Cache Penetration

When clients keep requesting keys that don't exist (`user:99999999`), every request hits the DB. This can happen by accident or by attack.

Fix: cache the miss.

```go
if !found {
    rdb.Set(ctx, key, "\x00__NIL__", 2*time.Minute) // short TTL
    return nil, ErrNotFound
}
// ... on read:
if val == "\x00__NIL__" {
    return nil, ErrNotFound
}
```

The `\x00` prefix makes collision with real data nearly impossible. Keep the TTL short — the key might get created later.

For really large key spaces, a **Bloom filter** in front of the cache is even better — if the filter says "definitely not here", skip both cache and DB.

### Proactive Refresh

Instead of waiting for a key to expire and making the next user pay the latency cost, refresh it in the background before it expires:

```go
remaining, _ := rdb.TTL(ctx, key).Result()
if remaining < ttl/5 {  // less than 20% TTL left
    go func() {
        fresh := db.Query(context.Background(), key)
        rdb.Set(context.Background(), key, fresh, ttl)
    }()
}
// return the current (still valid) cached value
```

The current request gets the cached value instantly. The refresh happens in the background. No one sees a cache miss.

In production you'll want to cap the number of concurrent refresh goroutines (a semaphore works well) and use a proper shutdown context instead of `context.Background()`.

### What to Monitor

| Metric | What to watch for |
|---|---|
| **Hit rate** | Below 95% on a read-heavy service? The cache might not be pulling its weight. (Write-heavy workloads will naturally be lower — measure per tier.) |
| **Invalidation latency** | CDC pipeline: < 2s. Pub/Sub: < 100ms. If it's climbing, check consumer lag. |
| **Eviction rate** | Should be near zero. If it's not, your `maxmemory` is too low. |
| **Penetration rate** | Requests for non-existent keys. Above 1%? Add negative caching or a Bloom filter. |

Tools: **Prometheus + Grafana** with the Redis Exporter. Key panels:

- `redis_keyspace_hits / (hits + misses)` — hit rate
- `rate(redis_evicted_keys_total[5m])` — evictions
- Custom app metrics for invalidation latency (instrument your CDC consumer)