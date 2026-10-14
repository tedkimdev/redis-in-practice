# Geospatial

Redis geospatial commands store longitude/latitude pairs and answer "what is near this point?" queries. Under the hood a geo index is just a **Sorted Set**: each member's score is a 52-bit **geohash** of its coordinates.

**What fits well**
- "Stores / drivers / restaurants near me"
- Simple geofencing and distance calculations

**Key trait**
- Because it's a Sorted Set, you can use `ZREM`, `ZCARD`, `ZRANGE`, etc. on a geo key.

## Commands

### GEOADD

```bash
# GEOADD key longitude latitude member  (longitude first!)
127.0.0.1:6379> GEOADD stores -79.3832 43.6532 "downtown" -79.4163 43.7615 "north-york" -79.6441 43.5890 "mississauga"
(integer) 3
```

### GEODIST & GEOPOS

```bash
127.0.0.1:6379> GEODIST stores downtown north-york km
"12.3..."

127.0.0.1:6379> GEOPOS stores downtown
1) 1) "-79.38320010900497437"
   2) "43.65320034187426069"
```

Coordinates come back slightly different from what you stored—the geohash encoding is lossy at the sub-meter level.

### GEOSEARCH

`GEOSEARCH` (Redis 6.2+) replaces the older `GEORADIUS` / `GEORADIUSBYMEMBER`.

```bash
# Stores within 15 km of a point, nearest first, with distance
127.0.0.1:6379> GEOSEARCH stores FROMLONLAT -79.3871 43.6426 BYRADIUS 15 km ASC WITHDIST
1) 1) "downtown"
   2) "1.2..."
2) 1) "north-york"
   2) "13.4..."

# Box search centered on a member
127.0.0.1:6379> GEOSEARCH stores FROMMEMBER downtown BYBOX 40 40 km ASC COUNT 5

# Store results into another key
127.0.0.1:6379> GEOSEARCHSTORE nearby:user:42 stores FROMLONLAT -79.3871 43.6426 BYRADIUS 5 km
```

(Distances above are illustrative; your output will show more decimals.)

## Practical patterns

### 1) Nearby search

Keep a static index per category (`stores`, `chargers`) and query with `GEOSEARCH ... COUNT N ASC`. Store details live in a Hash keyed by the member name.

### 2) Live positions (drivers, couriers)

Re-`GEOADD` the same member to update its position. Pair it with a per-driver heartbeat key with a TTL, because members inside a geo set **can't expire individually**—a sweeper removes members whose heartbeat is gone.

## Pitfalls

- **Longitude first.** `GEOADD key lon lat`—swapping them is the classic bug (and Redis rejects latitudes beyond ±85.05°).
- **Huge radii are slow.** `GEOSEARCH` is roughly O(N + log M) where N is the number of points in the searched area. A 1000 km radius over a dense dataset scans a lot; always use `COUNT`.
- **No polygons, no complex filters.** For "within this polygon AND open now AND rating > 4" use the Search module's geo fields or a real spatial database.

## Things to think about

1. Geo = Sorted Set + geohash scores.
2. Use `GEOSEARCH` (not `GEORADIUS`) for new code.
3. Members don't expire; manage liveness with separate TTL keys.
