# Bitmaps & Bitfields

Bitmaps and bitfields are not separate types—they are **commands that treat a String as an array of bits**. Because a String can be up to 512MB, a single key can address 2³² bits.

**What fits well**
- Per-user daily flags ("did user N log in today?")
- Feature flags and permission bits for dense integer IDs
- Many small packed counters in one key

**Key trait**
- One bit per entity: 10 million users fit in about **1.2MB**.

## Bitmap commands

```bash
# SETBIT key offset value — offset = user ID
127.0.0.1:6379> SETBIT login:2026-10-14 1001 1
(integer) 0
127.0.0.1:6379> SETBIT login:2026-10-14 1002 1
(integer) 0
127.0.0.1:6379> SETBIT login:2026-10-15 1001 1
(integer) 0

127.0.0.1:6379> GETBIT login:2026-10-14 1002
(integer) 1

# How many users logged in on the 14th?
127.0.0.1:6379> BITCOUNT login:2026-10-14
(integer) 2

# Users who logged in on BOTH days
127.0.0.1:6379> BITOP AND login:both login:2026-10-14 login:2026-10-15
(integer) 126
127.0.0.1:6379> BITCOUNT login:both
(integer) 1

# First user ID (bit) set to 1
127.0.0.1:6379> BITPOS login:2026-10-14 1
(integer) 1001
```

`BITOP` supports `AND`, `OR`, `XOR`, and `NOT`; its return value is the length in bytes of the destination string.

## BITFIELD

`BITFIELD` reads and writes **integers of arbitrary width** at arbitrary offsets inside one String—handy for packing many small counters.

```bash
# Two unsigned 8-bit counters; "#N" means "the N-th field of this width"
127.0.0.1:6379> BITFIELD stats:item:42 INCRBY u8 #0 1 INCRBY u8 #1 5
1) (integer) 1
2) (integer) 5

# Saturate instead of wrapping around at 255
127.0.0.1:6379> BITFIELD stats:item:42 OVERFLOW SAT INCRBY u8 #0 300
1) (integer) 255

127.0.0.1:6379> BITFIELD_RO stats:item:42 GET u8 #0 GET u8 #1
1) (integer) 255
2) (integer) 5
```

## Practical patterns

### 1) Daily active users and retention

One bitmap per day keyed by user ID. `BITCOUNT` gives DAU; `BITOP AND` across days gives "active on all N days"; `BITOP OR` across a week gives WAU.

### 2) Compact flags

`SETBIT user:flags:<feature> <user_id> 1` turns a feature on for a user in O(1) with negligible memory.

## Pitfalls

- **Sparse or huge IDs.** Setting bit 4,000,000,000 allocates ~500MB at once. Bitmaps only make sense for **dense** integer IDs; map UUIDs to sequential integers first, or use a Set / HyperLogLog instead.
- **`BITOP` and `BITCOUNT` are O(N)** on the string size. On big keys, run them on a replica or off-peak.
- **Large single keys.** One giant bitmap is a "big key" for replication and Cluster migration; shard by date or ID range.

## Things to think about

1. Bitmaps are Strings addressed bit by bit—great for dense IDs, terrible for sparse ones.
2. `BITCOUNT` + `BITOP` turn retention analytics into a few commands.
3. `BITFIELD` packs many small counters into one key with explicit overflow behavior.
