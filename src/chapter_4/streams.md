# Streams

A **Stream** is an append-only log stored under a single key. Each entry has an auto-generated, monotonically increasing ID (`<milliseconds>-<sequence>`) and a small set of field/value pairs—think "a Hash per entry, ordered by time."

Streams arrived in Redis 5.0 and fill the gap between **Lists** (simple queues, no replay, no fan-out with acknowledgement) and **Pub/Sub** (fan-out, but fire-and-forget).

**What fits well**
- Event logs and activity feeds
- Lightweight job queues that need acknowledgement and retries
- Fan-out of events to several independent consumer services

**Key trait**
- Entries are **persisted** in the key; readers track their own position, so you can replay history.

## Commands

### XADD & XRANGE

```bash
# Append: XADD key ID field value [field value ...]
# "*" lets Redis generate the ID
127.0.0.1:6379> XADD orders:events * type "created" order_id "789" amount "49.99"
"1760470000000-0"

127.0.0.1:6379> XADD orders:events * type "paid" order_id "789"
"1760470000150-0"

# Read a range: "-" is the smallest ID, "+" the largest
127.0.0.1:6379> XRANGE orders:events - +
1) 1) "1760470000000-0"
   2) 1) "type"
      2) "created"
      3) "order_id"
      4) "789"
      5) "amount"
      6) "49.99"
2) 1) "1760470000150-0"
   2) 1) "type"
      2) "paid"
      3) "order_id"
      4) "789"

127.0.0.1:6379> XLEN orders:events
(integer) 2
```

Think of a stream as a **CCTV recording**: you can watch the live feed, or rewind to see what happened yesterday at 2 PM.

You can also pass an explicit ID instead of `*`, but it must be greater than the last ID in the stream, so in practice you almost always let Redis generate it.

`XRANGE` takes `COUNT` to limit results, and `XREVRANGE key + -` walks from newest to oldest:

```bash
127.0.0.1:6379> XRANGE orders:events - + COUNT 10
127.0.0.1:6379> XREVRANGE orders:events + - COUNT 1
```

### XREAD (tail the stream)

```bash
# Wait (BLOCK 0 = forever) for entries newer than "$" (= anything added from now on)
127.0.0.1:6379> XREAD BLOCK 0 STREAMS orders:events $

# Wait at most 1s; returns (nil) if nothing arrived
127.0.0.1:6379> XREAD BLOCK 1000 STREAMS orders:events $

# Read from the beginning instead
127.0.0.1:6379> XREAD COUNT 100 STREAMS orders:events 0
```

`STREAMS` can list several keys at once (followed by one ID per key).

### Consumer groups

A consumer group lets several workers **share** a stream: each entry is delivered to one consumer in the group, and stays in the group's *Pending Entries List* (PEL) until it is acknowledged.

```bash
# Create the group, starting from the beginning ("0"); MKSTREAM creates the key if missing
127.0.0.1:6379> XGROUP CREATE orders:events billing 0 MKSTREAM
OK

# Worker "w1" reads new (never-delivered) entries: ">"
127.0.0.1:6379> XREADGROUP GROUP billing w1 COUNT 10 BLOCK 2000 STREAMS orders:events >

# Acknowledge after successful processing
127.0.0.1:6379> XACK orders:events billing 1760470000000-0
(integer) 1

# What is still pending (delivered but not acked)?
127.0.0.1:6379> XPENDING orders:events billing
```

If a worker dies, its pending entries can be taken over by another worker:

```bash
# Claim entries idle for more than 60s, scanning from ID 0
127.0.0.1:6379> XAUTOCLAIM orders:events billing w2 60000 0 COUNT 10
```

### Retention: reading does not delete

The most common misconception: *"Does a message disappear once a consumer reads it?"* **No.** Neither `XREAD` nor `XACK` removes entries. A stream behaves like Kafka's append-only log—entries stay until **you** remove them with a length cap, a time-based trim, or an explicit delete.

Streams grow forever unless you trim them.

```bash
# Keep roughly the last 100k entries ("~" lets Redis trim efficiently)
127.0.0.1:6379> XADD orders:events MAXLEN ~ 100000 * type "shipped" order_id "789"

# Or drop everything older than an ID — i.e. a time-based retention policy (Redis 6.2+)
127.0.0.1:6379> XTRIM orders:events MINID ~ 1760400000000-0

# Delete specific entries (rarely needed in practice)
127.0.0.1:6379> XDEL orders:events 1760470000000-0
(integer) 1
```

Because entry IDs start with a millisecond timestamp, `MINID` retention needs no extra "created_at" field.

## Streams vs Pub/Sub vs List

| | Pub/Sub | List (`LPUSH`/`BRPOP`) | Stream |
|---|---|---|---|
| Persistence | None — offline subscribers miss messages | Yes, until popped | Yes, until trimmed |
| Fan-out | Every subscriber | One consumer per item | Many groups, each sees every entry |
| Acknowledgement / retry | No | No (pop = gone) | Yes (`XACK`, PEL, `XAUTOCLAIM`) |
| Replay | No | No | Yes, by ID range |

Pub/Sub is still the right tool for ephemeral signals—cache invalidation broadcasts, presence, "something changed, go look." Redis 7.0 added **sharded Pub/Sub** (`SPUBLISH` / `SSUBSCRIBE`) so messages in a Cluster are routed only to the shard owning the channel instead of being broadcast to every node.

## Practical patterns

### 1) Reliable job queue

Producers `XADD`, workers `XREADGROUP ... >`, `XACK` on success, and a periodic `XAUTOCLAIM` sweeper re-delivers jobs from crashed workers. Check the delivery count in `XPENDING` and move poison messages to a dead-letter stream after N attempts.

### 2) Order and payment events (fan-out to services)

One stream, one consumer group per service. **Inside** a group, work is split between workers; **across** groups, every event is shared.

```
orders stream
 ├─ payment_group
 │    ├─ worker_1
 │    └─ worker_2
 └─ shipping_group
      ├─ worker_A
      └─ worker_B
```

```bash
127.0.0.1:6379> XADD orders * user_id 77 product_id 101 price 50000
"1760470200000-0"
127.0.0.1:6379> XGROUP CREATE orders payment_group 0
OK
127.0.0.1:6379> XGROUP CREATE orders shipping_group 0
OK
127.0.0.1:6379> XREADGROUP GROUP payment_group worker_1 COUNT 1 STREAMS orders >
127.0.0.1:6379> XACK orders payment_group 1760470200000-0
(integer) 1
```

If the payment service is down for maintenance, orders keep piling up safely in the stream; when it comes back, it resumes from where its group left off—no lost events.

### 3) Centralized log collection

Dozens of servers `XADD` error and access logs into one stream; later you `XRANGE` a specific time window to trace an incident.

### 4) Activity feed / audit log

`XADD ... MAXLEN ~ N` gives a capped, time-ordered log; `XREVRANGE key + - COUNT 20` returns the latest 20 entries.

## Pitfalls

- **Unbounded growth.** Forgetting `MAXLEN`/`MINID` is the #1 Streams memory incident.
- **Ever-growing PEL.** Entries that are read but never acked stay pending forever. Always ack, and monitor `XPENDING`.
- **At-least-once, not exactly-once.** A worker can crash after doing the work but before `XACK`; make handlers idempotent.
- **Not Kafka.** A stream is a single key living on a single shard, bounded by RAM. For very high throughput, partition across several stream keys yourself.

## Command summary

| Command | What it does | Notes |
|---|---|---|
| `XADD` | Append an entry | `*` auto-generates a timestamp ID |
| `XRANGE` / `XREVRANGE` | Read an ID range | History and time-window queries |
| `XREAD` | Read / block for new entries | `BLOCK` gives Pub/Sub-like waiting |
| `XLEN` | Number of entries | |
| `XGROUP` | Create/manage consumer groups | Required for distributed processing |
| `XREADGROUP` | Read as a group member | `>` = never-delivered entries |
| `XACK` | Mark an entry as processed | Removes it from the PEL, not the stream |
| `XTRIM` / `XDEL` | Trim or delete entries | Capacity management |

## Things to think about

1. Streams = persistent log + consumer groups; Pub/Sub = fire-and-forget fan-out. Use Pub/Sub for chat-style or lightweight notifications, Streams for orders, payments, and logs that must not be lost.
2. Trim every stream, ack every message, and design for redelivery.
3. Use one consumer group per downstream service for independent fan-out.
