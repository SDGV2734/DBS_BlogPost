---
title: "Key-Value Databases with Redis: A Complete Student Guide"
description: "Learn how Redis stores data, supports rich data structures, persists information, scales across nodes, and powers real-time applications."
author: "Sonam Dorji"
image:
  url: "../../assets/images/3.png"
  alt: "Illustration for the Redis key-value database guide"
pubDate: 2026-04-02
tags:
  ["Unit 2", "Redis", "Key-Value Database"]
---

# Introduction

Modern applications are expected to respond immediately. A social media feed must load in milliseconds, a game leaderboard must update in real time, and a shopping cart must remain available as a customer moves between pages.

Traditional relational databases remain essential for structured data and reliable transactions, but they are not always the best tool for extremely fast, key-based access. This is where **key-value databases** become useful.

In this Unit II guide, we will study **Redis**, one of the most widely used key-value databases. We will explore its architecture, data types, commands, persistence options, scaling methods, performance techniques, and security practices.

> **Key idea:** Redis does not replace PostgreSQL or MySQL. It usually complements them as a cache, session store, queue, leaderboard, or real-time data layer.

---

# 1. Introduction to Key-Value Databases

## What Is a Key-Value Database?

A key-value database stores information as pairs consisting of:

- a unique **key**, used to identify the data
- a **value**, containing the actual data

It works like a Python dictionary or Java `HashMap`:

```python
student = {
    "name": "Thinley",
    "age": 22
}
```

Redis uses the same basic idea:

```text
user:1001:name          -> "Alice"
session:abc123          -> '{"userId":1001,"role":"admin"}'
product:iphone15:price  -> "1299"
counter:visits          -> "94827"
```

A useful analogy is a gym locker system. The locker number is the key, and the belongings inside are the value. If we know the locker number, we can retrieve the contents without searching every locker.

## Redis Architecture

Applications communicate with a Redis server over a network connection, normally using port `6379`. The server parses commands, accesses its in-memory data structures, and returns a response.

```text
Web App ───────┐
Mobile App ────┼── TCP/IP ──> Redis Server
Microservice ──┘                 |
                                 ├── In-memory data structures
                                 ├── Persistence engine
                                 ├── Replication manager
                                 └── Pub/Sub engine
```

Redis primarily stores data in RAM and uses hash tables internally for key lookup. This gives common commands such as `GET` and `SET` an average time complexity of **O(1)**.

```text
SET username "Alice"
GET username
```

Because RAM access is much faster than disk access, Redis can provide very low latency and handle a large number of operations per second.

## Advantages and Limitations

Redis offers several important advantages:

| Advantage | Why It Matters |
|---|---|
| Fast access | In-memory operations avoid most disk I/O delays |
| Simple model | Data is accessed directly through unique keys |
| Rich data types | Lists, sets, hashes, sorted sets, and more |
| Atomic commands | Commands such as `INCR` safely update shared counters |
| TTL support | Temporary keys can expire automatically |
| Horizontal scaling | Redis Cluster distributes data across nodes |
| Pub/Sub support | Applications can broadcast real-time events |

Redis also has limitations:

| Limitation | Explanation |
|---|---|
| Memory-bound | RAM is more expensive than disk storage |
| No SQL-style joins | Data must be designed around known access patterns |
| Key-based access | Complex filtering requires additional structures or modules |
| Limited transactions | `MULTI`/`EXEC` does not provide SQL-style rollback |
| Persistence trade-offs | Durability requires careful RDB and AOF configuration |

---

# 2. Common Redis Use Cases

Redis is most useful when an application needs fast access to frequently changing or temporary data.

## Caching

Frequently requested database results or API responses can be stored in Redis:

```text
SETEX cache:homepage 300 "<html data>"
```

The value remains available for five minutes, reducing repeated work for the main database.

## Session Management

Login sessions can be stored with an automatic expiry:

```text
SETEX session:abc123 1800 "user:1001"
```

After 30 minutes, Redis removes the session automatically.

## Rate Limiting

Atomic counters and expiry times can track requests within a time window:

```text
INCR ratelimit:user:1001:api
EXPIRE ratelimit:user:1001:api 60
```

The application can reject additional requests after the counter reaches its limit.

## Leaderboards

Sorted sets maintain members in score order:

```text
ZADD leaderboard 9800 "player:alice"
ZADD leaderboard 8500 "player:bob"
ZREVRANGE leaderboard 0 9 WITHSCORES
```

This returns the ten highest-scoring players without sorting them at query time.

## Other Uses

Redis is also suitable for:

- task and message queues
- shopping carts
- live analytics
- distributed locks
- event broadcasting with Pub/Sub
- finding nearby locations
- autocomplete and search features

---

# 3. The Redis Data Model

Redis uses a flat key namespace. There are no tables, rows, or foreign keys, so consistent key naming is important.

## Key Naming Conventions

A common pattern is:

```text
object-type:id:attribute
```

Examples:

```text
user:1001:profile
user:1001:sessions
product:BTN500:details
order:ORD2024001:status
cache:homepage:trending
ratelimit:user:1001:api
```

Redis keys are case-sensitive, so `User:1001` and `user:1001` are different keys. Keys can be large, but short and readable names use less memory and are easier to manage.

---

# 4. Core Redis Data Types

Redis is more powerful than a basic key-value store because values can use specialized data structures.

## Strings

Strings store text, numbers, JSON, or binary data. They are useful for cached values, counters, and simple flags.

```text
SET username "Alice"
GET username

SET page:views 0
INCR page:views
INCRBY page:views 5

SETEX session:tok123 3600 "user:1001"
TTL session:tok123
```

Commands such as `INCR` are atomic, meaning concurrent clients cannot interrupt the update halfway through.

## Lists

Lists are ordered collections that allow duplicate values. Items can be added or removed from either end.

```text
LPUSH tasks "send_email"
LPUSH tasks "resize_image"
RPUSH tasks "generate_pdf"

LRANGE tasks 0 -1
LPOP tasks
RPOP tasks
```

Lists work well for queues and stacks. A worker can use a blocking command to wait for new jobs:

```text
BLPOP tasks 30
```

## Sets

Sets store unordered, unique values. Duplicate members are ignored.

```text
SADD tags:article:101 "redis" "nosql" "database"
SADD tags:article:102 "redis" "caching" "performance"

SMEMBERS tags:article:101
SINTER tags:article:101 tags:article:102
SUNION tags:article:101 tags:article:102
SDIFF tags:article:101 tags:article:102
```

Set operations are useful for tags, unique followers, permissions, and recommendation systems.

## Hashes

Hashes store field-value pairs under one key. They are useful for representing objects.

```text
HSET user:1001 name "Alice" email "alice@example.com" age 28 city "Thimphu"

HGET user:1001 name
HMGET user:1001 name email
HGETALL user:1001
HINCRBY user:1001 age 1
HDEL user:1001 city
```

Unlike storing an entire object as a JSON string, a hash allows one field to be updated without rewriting the whole object.

## Sorted Sets

A sorted set stores unique members together with numeric scores. Redis automatically keeps members ordered by score.

```text
ZADD leaderboard 9500 "player:alice"
ZADD leaderboard 8200 "player:bob"
ZADD leaderboard 9800 "player:charlie"

ZREVRANGE leaderboard 0 2 WITHSCORES
ZREVRANK leaderboard "player:alice"
ZINCRBY leaderboard 500 "player:alice"
```

Sorted sets are ideal for leaderboards, rankings, priority queues, and time-ordered data.

---

# 5. Specialized Redis Structures

## Bitmaps

Bitmaps treat a string as an array of bits. They efficiently record boolean information such as whether a user logged in on a particular day.

```text
SETBIT logins:2026-04-02 1001 1
SETBIT logins:2026-04-02 2005 1

GETBIT logins:2026-04-02 1001
BITCOUNT logins:2026-04-02
```

For millions of users, a bitmap can use far less memory than storing every user ID in a set.

## HyperLogLog

HyperLogLog estimates the number of unique items while using a small, fixed amount of memory.

```text
PFADD visitors:homepage "user:a" "user:b" "user:c"
PFADD visitors:homepage "user:a"
PFCOUNT visitors:homepage
```

It is useful for approximate unique visitor counts where a small estimation error is acceptable.

## Geospatial Indexes

Redis geospatial commands store coordinates and perform proximity searches.

```text
GEOADD drivers 89.6419 27.4712 "driver:D001"
GEOADD drivers 91.1006 26.1445 "driver:D002"

GEOPOS drivers "driver:D001"
GEODIST drivers "driver:D001" "driver:D002" km
GEOSEARCH drivers FROMLONLAT 89.6419 27.4712 BYRADIUS 50 km ASC
```

These commands can support nearby-driver, restaurant, store, and delivery-tracking features.

## Bloom Filters

A Bloom filter efficiently checks whether an item has probably been seen before:

- `0` means the item is definitely not present
- `1` means the item is probably present

Bloom filter commands require the RedisBloom module:

```text
BF.RESERVE crawled_urls 0.001 10000000
BF.ADD crawled_urls "https://example.com/page1"
BF.EXISTS crawled_urls "https://example.com/page1"
```

Bloom filters trade perfect accuracy for major memory savings.

---

# 6. Basic Commands and Safe Key Management

Global key commands work across Redis data types:

```text
EXISTS user:1001
TYPE user:1001
DEL user:1001
RENAME user:1001 user:old:1001

EXPIRE user:1001 3600
TTL user:1001
PERSIST user:1001
```

To search for keys, use cursor-based `SCAN`:

```text
SCAN 0 MATCH user:* COUNT 100
```

> **Production warning:** Avoid `KEYS *`. It scans the entire keyspace and can block Redis while it runs. `SCAN` performs incremental, non-blocking iteration.

Useful server commands include:

```text
PING
INFO
INFO memory
INFO replication
DBSIZE
SLOWLOG GET 10
```

Commands such as `FLUSHDB`, `FLUSHALL`, and `CONFIG SET` can make destructive changes and should be restricted.

---

# 7. Persistence and Durability

Redis stores its working dataset in memory. Persistence allows it to recover data after a restart.

## RDB Snapshots

RDB creates point-in-time snapshots of the dataset:

```text
save 900 1
save 300 10
save 60 10000
```

Manual snapshot commands include:

```text
BGSAVE
LASTSAVE
```

RDB files are compact and fast to restore, but changes made after the most recent snapshot may be lost.

## Append-Only File

AOF records write operations so Redis can replay them during startup:

```text
appendonly yes
appendfsync everysec
```

Common synchronization policies are:

| Policy | Trade-off |
|---|---|
| `always` | Highest durability, lower write performance |
| `everysec` | Balanced option with up to about one second of data loss |
| `no` | Lets the operating system decide when to sync |

## Choosing a Persistence Strategy

| Strategy | Best Use |
|---|---|
| No persistence | Rebuildable, temporary cache |
| RDB only | Backups and fast recovery where some loss is acceptable |
| AOF only | Workloads requiring greater durability |
| RDB and AOF | Balanced production setup |

---

# 8. High Availability and Scaling

## Replication

Redis replicas maintain copies of data from a primary server:

```text
REPLICAOF 192.168.1.100 6379
INFO replication
```

Replication improves read capacity and provides copies that can take over after a failure.

## Redis Sentinel

Sentinel monitors Redis instances and performs automatic failover. If the primary becomes unavailable, Sentinel can promote a replica.

```text
Primary ──replicates──> Replica 1
   │
   └──────replicates──> Replica 2

Sentinel nodes monitor the primary and replicas.
```

Sentinel improves **high availability**, but the full dataset must still fit on one Redis server.

## Redis Cluster

Redis Cluster provides horizontal scaling by distributing data across **16,384 hash slots**.

```text
Node A: slots 0-5460
Node B: slots 5461-10922
Node C: slots 10923-16383
```

Each key is assigned to a slot:

```text
slot = CRC16(key) % 16384
```

Hash tags force related keys into the same slot:

```text
{user:1001}.name
{user:1001}.email
```

| Feature | Sentinel | Cluster |
|---|---|---|
| Main purpose | Automatic failover | Scaling and high availability |
| Dataset size | Limited by one server | Combined capacity of cluster nodes |
| Write throughput | Limited by primary | Distributed across primary nodes |
| Multi-key commands | Fully supported | Keys must share a hash slot |

---

# 9. Redis Modules and Extensions

Redis modules add specialized capabilities beyond core Redis:

- **RediSearch** adds full-text search, filtering, and aggregation.
- **RedisJSON** stores and partially updates JSON documents.
- **RedisTimeSeries** stores timestamped measurements and aggregations.
- **RedisBloom** adds Bloom and Cuckoo filters.

For example, RedisJSON can update one nested field without rewriting an entire JSON document:

```text
JSON.SET user:1001 $ '{"name":"Alice","address":{"city":"Thimphu"}}'
JSON.GET user:1001 $.address.city
JSON.SET user:1001 $.address.city '"Paro"'
```

Module commands are only available when the relevant module is installed.

---

# 10. Performance Optimization

## Use Pipelining

Every command normally requires a network round trip. Pipelining sends many commands together:

```python
import redis

r = redis.Redis()
pipe = r.pipeline()

for i in range(10000):
    pipe.set(f"key:{i}", f"value:{i}")

results = pipe.execute()
```

This can significantly improve bulk-operation performance.

## Understand Redis Transactions

`MULTI` and `EXEC` queue commands and execute them without other commands being interleaved:

```text
MULTI
DECRBY user:1001:credits 100
INCRBY user:2001:credits 100
EXEC
```

Redis does not roll back successful commands when another queued command fails at runtime. For complex atomic logic, Lua scripts can perform checks and updates as one operation.

## Configure Memory and Eviction

Redis needs a memory limit and an eviction policy:

```text
maxmemory 4gb
maxmemory-policy allkeys-lru
```

Common policies include:

| Policy | Behavior |
|---|---|
| `noeviction` | Reject writes when memory is full |
| `allkeys-lru` | Remove least recently used keys |
| `allkeys-lfu` | Remove least frequently used keys |
| `volatile-lru` | Remove least recently used keys that have a TTL |
| `volatile-ttl` | Remove keys that will expire soonest |

Monitor Redis using:

```text
INFO stats
INFO memory
SLOWLOG GET 25
MEMORY USAGE user:1001
MEMORY DOCTOR
```

---

# 11. Redis Security

Redis should be treated as an internal service and should never be exposed openly to the internet.

## Use Access Control Lists

Redis 6 and later support ACL users with restricted commands and key patterns:

```text
ACL SETUSER api_service on >strong_password ~cache:* +GET +MGET +SETEX
ACL SETUSER default off
ACL LIST
ACL LOG
```

This is safer than giving every application full access.

## Security Checklist

- bind Redis only to trusted network interfaces
- keep protected mode enabled
- use a firewall to restrict Redis ports
- create least-privilege ACL users
- use strong passwords and TLS
- run Redis as a non-root user
- restrict dangerous administrative commands
- set `maxmemory` to reduce out-of-memory risk
- monitor ACL logs, slow logs, and server metrics

Example network configuration:

```text
bind 127.0.0.1 10.0.0.50
protected-mode yes
```

> Authentication alone is not enough. Redis security should combine network isolation, authorization, encryption, command restrictions, and monitoring.

---

# Quick Reference

| Data Type | Common Commands | Typical Use |
|---|---|---|
| String | `SET`, `GET`, `INCR`, `SETEX` | Caching, counters, sessions |
| List | `LPUSH`, `RPUSH`, `LPOP`, `BLPOP` | Queues and stacks |
| Set | `SADD`, `SMEMBERS`, `SINTER` | Unique collections |
| Hash | `HSET`, `HGET`, `HGETALL` | Objects and profiles |
| Sorted Set | `ZADD`, `ZRANGE`, `ZREVRANK` | Rankings and priority queues |
| Bitmap | `SETBIT`, `GETBIT`, `BITCOUNT` | Boolean activity tracking |
| HyperLogLog | `PFADD`, `PFCOUNT` | Approximate unique counts |
| Geospatial | `GEOADD`, `GEODIST`, `GEOSEARCH` | Nearby-location searches |

## Complexity Reference

| Operation | Average Complexity |
|---|---|
| `GET`, `SET`, `HGET`, `HSET`, `SADD` | O(1) |
| `LPUSH`, `RPUSH`, `LPOP`, `RPOP` | O(1) |
| `ZADD`, `ZRANK` | O(log N) |
| `HGETALL` | O(N) |
| `KEYS *` | O(N) and blocking |
| `SCAN` | O(1) per call, O(N) for complete iteration |

---

# Key Takeaways

- Redis is an in-memory key-value database designed for fast access.
- Its rich data types solve different problems without SQL-style joins.
- Keys should follow a consistent naming convention such as `type:id:field`.
- Use TTLs for temporary data and `SCAN` instead of `KEYS *` in production.
- RDB and AOF provide different durability and recovery trade-offs.
- Sentinel provides failover, while Redis Cluster provides scaling and high availability.
- Pipelining, suitable data types, memory limits, and monitoring improve performance.
- Redis must be protected through network controls, ACLs, TLS, and restricted commands.

Redis is most effective when used for the right workload. By understanding its data structures and operational trade-offs, we can use it alongside relational databases to build responsive, scalable, and reliable applications.
