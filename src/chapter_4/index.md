# Chapter 4: Recent Trends and Advanced Structures

## Intro

This chapter looks at **where Redis is heading**—features, deployment patterns, and ecosystem shifts you see in production today—and at **advanced capabilities** beyond the basic types from Chapter 3.

Chapter 3 covered the "everyday" types: String, List, Set, Hash, and Sorted Set. Most applications never need more. But once you hit problems like event logs, counting billions of unique visitors, "find stores near me," or semantic search, the advanced structures save you from reinventing them badly on top of Strings and Hashes.

## What's in this chapter

**Advanced core structures**
- [Streams](./streams.md) — append-only logs with consumer groups (and how they compare to Pub/Sub)
- [Bitmaps & Bitfields](./bitmaps_and_bitfields.md) — bit-level tricks for flags and compact counters
- [HyperLogLog](./hyperloglog.md) — approximate unique counts in 12KB
- [Geospatial](./geospatial.md) — radius and box queries on coordinates

**Beyond the core**
- [JSON, Search & Probabilistic Structures](./json_search_probabilistic.md) — what used to be "Redis Stack"
- [Vector Search](./vector_search.md) — Redis as a vector store for AI workloads

**Ecosystem**
- [Redis 7.x/8, the License Change, and Valkey](./ecosystem_trends.md) — versions, licensing, and the fork

> **Version note:** Commands in this chapter were checked against Redis 7.2+ unless a section says otherwise. Features that need Redis 8 (or the old Redis Stack modules) are called out explicitly.
