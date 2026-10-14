# HyperLogLog

**HyperLogLog (HLL)** is a probabilistic structure that estimates the number of **unique** elements in a set. It never stores the elements themselves, so it uses at most about **12KB per key** no matter how many items you add, with a standard error of **0.81%**.

**What fits well**
- Unique visitors / unique searches / unique IPs per day
- Any "count distinct" where ±1% is acceptable

**Key trait**
- You can add and count, but you **cannot** list members or check whether a specific member was added.

## Commands

```bash
# PFADD returns 1 if the internal estimate changed
127.0.0.1:6379> PFADD uv:2026-10-14 "user:1" "user:2" "user:3"
(integer) 1
127.0.0.1:6379> PFADD uv:2026-10-14 "user:2"
(integer) 0

127.0.0.1:6379> PFCOUNT uv:2026-10-14
(integer) 3

127.0.0.1:6379> PFADD uv:2026-10-15 "user:3" "user:4"
(integer) 1

# Union: unique users across both days
127.0.0.1:6379> PFCOUNT uv:2026-10-14 uv:2026-10-15
(integer) 4

# Persist the union into a new key
127.0.0.1:6379> PFMERGE uv:2026-w42 uv:2026-10-14 uv:2026-10-15
OK
```

(The `PF` prefix honors Philippe Flajolet, who invented the algorithm.)

## Set vs HyperLogLog

| | Set | HyperLogLog |
|---|---|---|
| Memory for 10M uniques | Hundreds of MB | ~12KB |
| Accuracy | Exact | ~0.81% standard error |
| Membership check / list members | Yes | No |
| Union | `SUNION` (expensive) | `PFMERGE` (cheap) |

## Practical patterns

### 1) Daily / weekly / monthly uniques

`PFADD` into a daily key, then `PFMERGE` daily keys into weekly and monthly ones. Unions are lossless for HLL—you don't need to re-scan raw events.

### 2) Per-entity unique counts

`PFADD article:42:readers <user_id>` for millions of articles stays affordable because small HLLs use a compact sparse encoding.

## Pitfalls

- **No intersections.** HLL supports union, not intersection. Estimating "A ∩ B" via inclusion–exclusion compounds the error badly for small overlaps.
- **Not for billing or exact numbers.** If someone gets paid per unique, use an exact structure.
- **Small counts.** At very small cardinalities the estimate is close to exact, but don't build logic that depends on it being exact.

## Things to think about

1. HLL trades ~1% error for constant ~12KB memory.
2. Use `PFMERGE` to roll daily counts up to longer periods.
3. If you need membership checks, look at Sets or Bloom filters instead.
