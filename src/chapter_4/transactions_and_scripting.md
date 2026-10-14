# Transactions & Scripting

Atomic commands like `GETDEL` or `HSETEX` cover many cases, but real business logic often needs **"read, decide, then write"**—and that can't be expressed in one command. This section covers the three tools Redis gives you: `MULTI`/`EXEC`/`WATCH`, Lua scripts (`EVAL`), and Redis Functions.

## The problem: a first-come coupon

Issue a coupon only if stock remains: `GET` the count, check `> 0`, then `DECR`. Done from application code, two clients can interleave:

```
Client A: GET coupon -> 1
Client B: GET coupon -> 1
Client A: DECR       -> 0
Client B: DECR       -> -1   ❌ over-issued
```

This is a classic race condition. The read, the check, and the write must run as **one indivisible unit** inside Redis.

## MULTI / EXEC / WATCH

- `MULTI` starts a transaction; subsequent commands are **queued**, not executed.
- `EXEC` runs the queue in order; no other client's command can run in between.
- `DISCARD` throws the queue away.
- `WATCH key` adds **optimistic locking**: if a watched key changes before `EXEC`, the transaction is aborted.

```bash
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379(TX)> SET item:100:stock 10
QUEUED
127.0.0.1:6379(TX)> INCR item:100:sales
QUEUED
127.0.0.1:6379(TX)> EXEC
1) OK
2) (integer) 1
```

### WATCH for optimistic locking

```bash
127.0.0.1:6379> SET coupon:100 1
OK
127.0.0.1:6379> WATCH coupon:100
OK
127.0.0.1:6379> GET coupon:100
"1"

# ... meanwhile another client runs: DECR coupon:100

127.0.0.1:6379> MULTI
OK
127.0.0.1:6379(TX)> DECR coupon:100
QUEUED
127.0.0.1:6379(TX)> EXEC
(nil)    # watched key changed -> transaction aborted, no over-issue
```

### Limitations

- **No conditionals inside the transaction.** Queued commands don't return values until `EXEC`, so you can't branch on them.
- **No rollback.** If one queued command fails at runtime (e.g. wrong type), the others still run.
- **WATCH means retries.** Under contention, the application must loop and retry aborted transactions.

## Lua scripts (EVAL)

Redis embeds a **Lua 5.1** interpreter. You send the whole script; since commands execute one at a time, **nothing else runs while the script runs**, so the entire script is atomic—branches included.

```
EVAL "<lua code>" <numkeys> key1 key2 ... arg1 arg2 ...
```

- `KEYS[1]`, `KEYS[2]`, … — key names passed in
- `ARGV[1]`, `ARGV[2]`, … — other arguments
- `redis.call('CMD', ...)` — run a Redis command; an error aborts the script
- `redis.pcall('CMD', ...)` — same, but returns the error so the script can continue

```bash
127.0.0.1:6379> EVAL "return 'Hello Lua!'" 0
"Hello Lua!"
127.0.0.1:6379> EVAL "return KEYS[1] .. ' and ' .. ARGV[1]" 1 mykey myval
"mykey and myval"
```

### The coupon, atomically

```lua
local stock = tonumber(redis.call('GET', KEYS[1]))
if stock and stock > 0 then
    redis.call('DECR', KEYS[1])
    return 1
else
    return 0
end
```

```bash
127.0.0.1:6379> SET coupon:100 2
OK
127.0.0.1:6379> EVAL "local stock = tonumber(redis.call('GET', KEYS[1])); if stock and stock > 0 then redis.call('DECR', KEYS[1]); return 1 else return 0 end" 1 coupon:100
(integer) 1
127.0.0.1:6379> EVAL "..." 1 coupon:100
(integer) 1
127.0.0.1:6379> EVAL "..." 1 coupon:100
(integer) 0    # out of stock, never goes negative
```

In application code, keep scripts in multi-line strings or separate `.lua` files. Most clients wrap `SCRIPT LOAD` + `EVALSHA` so the script body is sent once and then referenced by its SHA1 hash (in go-redis, `redis.NewScript(...).Run(...)`, as used in Chapter 6).

## Redis Functions (7.0+)

`EVAL` scripts live in your app: they are re-sent (or re-loaded) by every client, hard to share across services, and hard to version. **Functions** store Lua code **on the server** as named functions—similar to a database stored procedure. The language is still Lua; what changes is how code is deployed and managed.

### 1. Write a library

```lua
#!lua name=mylib

local function deduct_stock(keys, args)
    local amount = tonumber(args[1])
    local user = args[2]
    local stock = tonumber(redis.call('GET', keys[1]))
    if stock and stock >= amount then
        redis.call('DECRBY', keys[1], amount)
        return user .. ": success (stock left: " .. (stock - amount) .. ")"
    else
        return user .. ": fail (out of stock)"
    end
end

redis.register_function('deduct_stock', deduct_stock)
```

The `#!lua name=mylib` shebang line is required: it declares the engine and the library name.

### 2. Load it

```bash
# -x passes stdin as the last argument
cat coupon_logic.lua | redis-cli -x FUNCTION LOAD REPLACE

# Redis running in Docker (use -i, not -it, so stdin redirection works)
docker exec -i my-redis redis-cli -x FUNCTION LOAD REPLACE < coupon_logic.lua
```

### 3. Call it

```bash
127.0.0.1:6379> SET coupon:200 10
OK
127.0.0.1:6379> FCALL deduct_stock 1 coupon:200 3 "user_A"
"user_A: success (stock left: 7)"
127.0.0.1:6379> FCALL deduct_stock 1 coupon:200 8 "user_B"
"user_B: fail (out of stock)"
```

Update with `FUNCTION LOAD REPLACE`, remove with `FUNCTION DELETE mylib`, inspect with `FUNCTION LIST`.

### EVAL vs Functions: persistence

| | `EVAL` scripts | Functions |
|---|---|---|
| Stored in | Script cache (memory only) | The dataset itself |
| After restart | Gone—clients must reload | Still there (saved in RDB/AOF) |
| Replication | Script effects replicate | The library replicates to replicas |
| Caller needs | The full script (or its SHA) | Just the function name |

Load functions once from your deploy pipeline or startup scripts; application code then only calls `FCALL deduct_stock ...`.

## Pitfalls

- **Scripts block the server.** While a script or function runs, every other client waits. No long loops, no scanning big keys—keep them short. If one runs past `busy-reply-threshold` (formerly `lua-time-limit`, default 5s), stop it with `SCRIPT KILL` (for `EVAL`) or `FUNCTION KILL` (for functions); either works only if the script hasn't written anything yet—otherwise only `SHUTDOWN NOSAVE` helps.
- **Declare every key in `KEYS`.** Building key names inside the script breaks Redis Cluster routing; all keys in one call must hash to the same slot (use hash tags like `{user:42}`).
- **`MULTI` is not rollback.** Validate inputs before queuing commands.
- **Functions are data.** `FLUSHALL` does not remove them (use `FUNCTION FLUSH` for that), but restoring an RDB from another environment can bring in—or miss—libraries. Version your `.lua` files in Git.

## Things to think about

1. `MULTI`/`EXEC` batches commands atomically but can't branch; `WATCH` detects conflicts at the cost of retries.
2. Lua (`EVAL`) runs read–decide–write logic as one atomic operation.
3. Functions (7.0+) keep that logic on the server, persisted and replicated, called by name.
4. Keep server-side code tiny: it runs on the same thread as everything else.
