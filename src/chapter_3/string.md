# String

The **String** type is Redis’s most fundamental value type. You can store text, binary payloads, and serialized blobs (for example JSON) under a single key.

Despite the name “string,” Redis Strings are effectively **byte sequences** and can be up to **512MB** per value.

**What fits well**
- Plain text and serialized objects (JSON, protobuf, etc.)
- Small binary payloads
- Numeric counters (via `INCR` / `DECR` family commands)

**Key trait**
- One key maps to **one** value (the simplest model in Redis).

> **Note:** Avoid huge values in production. Even though 512MB is the hard limit, very large values waste RAM, increase fragmentation risk, and can hurt latency.

## Commands

### SET & GET

```bash
# Store: SET key value
127.0.0.1:6379> SET user:42:email "alice@corp.io"
OK

# Read: GET key
127.0.0.1:6379> GET user:42:email
"alice@corp.io"
```

### INCR & DECR

`INCR`/`DECR` work on **integer** string values.

```bash
127.0.0.1:6379> SET article:7:likes 250
OK

127.0.0.1:6379> INCR article:7:likes
(integer) 251

127.0.0.1:6379> INCRBY article:7:likes 5
(integer) 256

127.0.0.1:6379> DECR article:7:likes
(integer) 255

127.0.0.1:6379> DECRBY article:7:likes 3
(integer) 252
```

> **Float counters:** `INCR`/`DECR` only work on integer strings — calling `INCR` on `"3.14"` returns an error. Use `INCRBYFLOAT` for decimal values (there is no `DECRBYFLOAT`; pass a negative value instead).

### MSET & MGET

```bash
127.0.0.1:6379> MSET user:42:name "Alice" user:42:role "admin"
OK

127.0.0.1:6379> MGET user:42:name user:42:role user:42:bio
1) "Alice"
2) "admin"
3) (nil)
```

Missing keys return `(nil)` — the response preserves positional order, so your application can match each result back to the requested key by index.

## Practical patterns

### 1) Simple caching

Store a DB/API response as a string value (often JSON).

```bash
SET product:56 "{\"id\":56,\"name\":\"Wireless Keyboard\",\"price\":49.99}"
```

### 2) Counters

Use cases include page views, likes, daily API quotas, and so on.

`INCR` is atomic as a single Redis command, which is much safer than read-modify-write loops in application code under concurrency.

### 3) Short-lived verification codes

Store a 6-digit SMS code and expire it automatically with `EX` (TTL is covered in [Expire & TTL](./expire_and_ttl.md); for the full list of SET options see [SET Command & Options](./set_command.md)).

```bash
SET auth:phone:821098765432 "482901" EX 300
```

`EX 300` sets expiry to **300 seconds**.

### 4) Distributed lock (basic pattern)

Use `NX` so the key is set only if it does not exist, and `EX` to auto-expire the lock if a process crashes.

```bash
SET lock:order:789 "worker-a1b2" NX EX 10
```

> **Production note:** Real distributed locks usually need token-safe unlock rules so you do not delete another holder’s lock. Treat this as the simplest Redis building block, not the full solution.

## Go example

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/redis/go-redis/v9"
)

func main() {
	ctx := context.Background()

	rdb := redis.NewClient(&redis.Options{
		Addr: "localhost:6379",
	})
	defer rdb.Close() // always close in real apps

	pong, err := rdb.Ping(ctx).Result()
	if err != nil {
		log.Fatal("Redis connection failed: ", err)
	}
	fmt.Println(pong)

	// SET — 0 means no expiry (time.Duration)
	err = rdb.Set(ctx, "user:42:email", "alice@corp.io", 0).Err()
	if err != nil {
		log.Fatal(err)
	}

	// GET
	val, err := rdb.Get(ctx, "user:42:email").Result()
	if err == redis.Nil {
		fmt.Println("key does not exist")
	} else if err != nil {
		log.Fatal(err)
	} else {
		fmt.Println("user:42:email ->", val)
	}

	// SET counter
	rdb.Set(ctx, "article:7:likes", 250, 0)

	// INCR
	n, err := rdb.Incr(ctx, "article:7:likes").Result()
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("after INCR:      ", n) // 251

	// INCRBY
	n, err = rdb.IncrBy(ctx, "article:7:likes", 5).Result()
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("after INCRBY 5:  ", n) // 256

	// DECR
	n, err = rdb.Decr(ctx, "article:7:likes").Result()
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("after DECR:      ", n) // 255

	// DECRBY
	n, err = rdb.DecrBy(ctx, "article:7:likes", 3).Result()
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("after DECRBY 3:  ", n) // 252

	// MSET
	err = rdb.MSet(ctx, "user:42:name", "Alice", "user:42:role", "admin").Err()
	if err != nil {
		log.Fatal(err)
	}

	// MGET
	vals, err := rdb.MGet(ctx, "user:42:name", "user:42:role", "user:42:bio").Result()
	if err != nil {
		log.Fatal(err)
	}

	keys := []string{"user:42:name", "user:42:role", "user:42:bio"}
	for i, v := range vals {
		fmt.Printf("%s -> %v\n", keys[i], v)
	}
}
```

## Things to think about

1. String is Redis’s foundational type for text/binary payloads stored as a single blob per key.
2. The maximum String size is **512MB**—a limit, not a target.
3. `INCR`/`DECR` are a practical way to implement safe integer counters under concurrency.
4. Consistent key naming (often with `:` segments like `user:42:email`) makes large keyspaces operable.
