# What is Redis?

## Remote Dictionary Server

**Redis** stands for **REmote DIctionary Server**.

Redis is a **data structure server**. It knows whether your data is a List, Set, or Sorted Set, and it can run operations directly on those structures on the server side.

That means you do not need to fetch entire values and process everything in application code.  
For example, you can:
- pop only the last item from a list,
- check whether a set contains a member,
- update specific fields in a hash.

Redis is not just a blob-based key-value store. It supports **data-structure-level operations** inside the server.

## Reference: What is Memcached?

Before Redis became dominant, **Memcached** was the most widely used in-memory cache.

- **Characteristics:** simple and fast; stores values as raw byte blobs in a basic key-value model.
- **Strength:** supports multithreading and is very fast for simple read-heavy workloads.
- **Limitations:** no rich data types, fully volatile (no persistence in common setups), and limited built-in replication compared to Redis.

Memcached is still used for simple caching, but Redis is now the mainstream choice for modern backends that need richer capabilities.

## Redis data types: choosing the right tool

Redis provides multiple data types, and you choose based on your business requirement:

| Data Type | Description | Use case |
|---|---|---|
| **Strings** | Basic text/number storage | Session tokens, counters |
| **Lists** | Ordered sequence by insertion order | Task queues, recent activity |
| **Sets** | Unordered unique collection (no duplicates) | Tags, unique visitors |
| **Hashes** | Field-value object structure | User profiles, config |
| **Sorted Sets** | Uniques automatically sorted by score | Leaderboards, rate limiting |

A wider set of data structures means Redis can handle logic like sorting, uniqueness checks, and rank queries for you.  
You mainly focus on selecting the right structure.

## Redis design intent: keep data and operations together

Redis is designed to keep **data and operations together**. By offering predefined data structures with atomic commands, it simplifies application logic and reduces unnecessary network round-trips.

## Summary

1. Redis is more than a simple cache; it is a **Remote Dictionary Server**.
2. Redis is a **data structure server** that understands data shape.
3. Choosing the right Redis data type can significantly reduce both network cost and backend code complexity.