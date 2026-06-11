---
title: "Apache Cassandra: A Complete Guide to Column-Family Databases"
description: "Learn Cassandra architecture, query-driven data modeling, CQL, consistency levels, replication, read and write paths, repair, and compaction."
author: "Sonam Dorji"
image:
  url: "../../assets/images/1.png"
  alt: "Illustration for the Apache Cassandra column-family database guide"
pubDate: 2026-05-21
tags:
  ["Unit 4", "Cassandra", "Column-Family Database"]
---

# Introduction

Imagine operating a global messaging platform where millions of users continuously send messages. The database must accept heavy write traffic, serve users from several regions, and remain available even when servers fail.

Apache Cassandra was created for this kind of problem.

**Cassandra** is an open-source, distributed column-family database designed for high availability, horizontal scalability, fault tolerance, and write-heavy workloads. It uses a peer-to-peer architecture with no single master node, allowing a cluster to continue operating when individual nodes become unavailable.

This Unit IV guide explores Cassandra's architecture, wide-row data model, CQL, query-driven schema design, consistency levels, replication, read and write paths, repair, and compaction.

> **Core idea:** Cassandra is not a relational database with different syntax. It requires tables to be designed around specific application queries.

---

# 1. Why Cassandra?

## Distributed and Decentralized Architecture

In Cassandra, every node is an equal peer. There is no permanent master responsible for all operations.

```text
          Node A
       /          \
   Node D          Node B
       \          /
          Node C
```

Any node can receive a client request and act as the **coordinator** for that request. The coordinator determines which nodes contain the required replicas and communicates with them.

This design provides:

- no single point of failure
- continuous operation during node failures
- simpler horizontal scaling
- distributed reads and writes

## Elastic Scalability

Cassandra scales horizontally by adding nodes to the cluster. New nodes take responsibility for part of the token range, increasing storage capacity and throughput.

Unlike scaling up a single database server, scaling out spreads workload across many machines.

## High Availability and Replication

Cassandra stores copies of each partition on multiple nodes. The **replication factor** determines the number of copies.

```text
Replication factor = 3

Partition copy 1 -> Node A
Partition copy 2 -> Node C
Partition copy 3 -> Node D
```

If one replica is unavailable, other replicas can continue serving requests.

## Tunable Consistency

Cassandra allows each read or write to specify how many replicas must respond before the operation succeeds.

- Fewer required responses improve availability and latency.
- More required responses improve consistency but may reduce availability.

This per-operation choice is called **tunable consistency**.

## Suitable Workloads

Cassandra is a strong choice for:

- time-series and IoT data
- logs and application events
- messaging and activity feeds
- write-heavy analytics
- globally distributed applications
- workloads requiring continuous availability

Cassandra is usually a poor fit for applications requiring joins, foreign keys, complex ad hoc filtering, or multi-row relational transactions.

---

# 2. CAP Theorem and Cassandra

The CAP theorem describes three important properties of distributed systems:

| Property | Meaning |
|---|---|
| Consistency | Every read receives the latest successful write |
| Availability | Every request receives a response |
| Partition tolerance | The system continues operating during network partitions |

During a network partition, a distributed database must make trade-offs between consistency and availability.

Cassandra is commonly described as an **AP-oriented system** because it prioritizes availability and partition tolerance. However, its consistency levels allow applications to request stronger consistency when necessary.

> Cassandra offers a dial rather than one fixed consistency mode.

---

# 3. Cassandra Cluster Architecture

## Cluster Topology

Cassandra organizes infrastructure using this hierarchy:

```text
Cluster
└── Data Center
    └── Rack
        └── Node
```

- A **cluster** contains all cooperating Cassandra nodes.
- A **data center** groups nodes by region or deployment location.
- A **rack** represents a failure boundary within a data center.
- A **node** is one Cassandra process storing part of the data.

Topology-aware replication places copies across different racks or data centers so one hardware failure does not remove every replica.

## Rings, Tokens, and Consistent Hashing

Cassandra maps partition keys and nodes into a numeric token space.

```text
partition key -> hash function -> token -> responsible nodes
```

The default `Murmur3Partitioner` hashes a partition key to determine its token. Nodes own ranges of tokens and store partitions whose tokens fall within those ranges.

**Consistent hashing** limits how much data must move when nodes are added or removed.

## Virtual Nodes

Virtual nodes, or **vnodes**, allow one physical node to own many smaller token ranges.

Benefits include:

- more even data distribution
- easier node replacement
- faster cluster expansion
- automatic redistribution of ranges

## Gossip Protocol

Nodes exchange cluster-state information through the gossip protocol. Each node periodically shares what it knows with other nodes, allowing cluster membership and health information to spread.

Cassandra combines gossip with a failure detector to estimate whether another node is reachable.

## Snitches

A **snitch** tells Cassandra which nodes belong to each data center and rack. Cassandra uses this topology information when placing replicas and routing requests.

For production deployments, `GossipingPropertyFileSnitch` is a commonly used topology configuration.

---

# 4. Replication Strategies

A keyspace defines how Cassandra places replicas.

## SimpleStrategy

`SimpleStrategy` places replicas without awareness of racks or multiple data centers. It is suitable only for basic development and testing.

```sql
CREATE KEYSPACE learning
WITH replication = {
  'class': 'SimpleStrategy',
  'replication_factor': 1
};
```

## NetworkTopologyStrategy

`NetworkTopologyStrategy` supports rack-aware and multi-data-center deployments.

```sql
CREATE KEYSPACE ecommerce
WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'DC1': 3,
  'DC2': 3
};
```

This configuration stores three replicas in each data center.

> Use `NetworkTopologyStrategy` for production deployments, including single-data-center production clusters.

---

# 5. Cassandra's Wide-Row Data Model

Cassandra is a column-family database with a wide-row model. Its tables resemble relational tables, but their behavior and design rules are different.

```text
Cluster -> Keyspace -> Table -> Partition -> Rows and columns
```

## Keyspaces

A **keyspace** is the outermost data container, similar to a database in an RDBMS. It defines replication settings for its tables.

## Tables

Create a basic table using Cassandra Query Language:

```sql
CREATE TABLE users (
  user_id UUID,
  email TEXT,
  username TEXT,
  created_at TIMESTAMP,
  PRIMARY KEY (user_id)
);
```

## Primary Key Anatomy

A Cassandra primary key contains:

1. A **partition key**, which determines where data is stored.
2. Optional **clustering columns**, which determine how rows are sorted inside a partition.

```sql
PRIMARY KEY ((partition_key), clustering_column_1, clustering_column_2)
```

Example:

```sql
CREATE TABLE posts_by_author (
  author_id UUID,
  created_at TIMESTAMP,
  post_id UUID,
  title TEXT,
  body TEXT,
  PRIMARY KEY ((author_id), created_at, post_id)
) WITH CLUSTERING ORDER BY (created_at DESC);
```

Here:

- `author_id` selects the partition.
- `created_at` and `post_id` order rows within that partition.
- posts for one author can be returned newest-first without sorting at query time.

## CQL Data Types

Common CQL types include:

| Category | Examples |
|---|---|
| Identifiers | `UUID`, `TIMEUUID` |
| Text | `TEXT`, `VARCHAR` |
| Numbers | `INT`, `BIGINT`, `FLOAT`, `DOUBLE`, `DECIMAL` |
| Date and time | `TIMESTAMP`, `DATE`, `TIME` |
| Other simple types | `BOOLEAN`, `BLOB` |
| Collections | `LIST`, `SET`, `MAP` |

User-defined types can group related fields:

```sql
CREATE TYPE address (
  street TEXT,
  city TEXT,
  postal_code TEXT
);

CREATE TABLE customers (
  customer_id UUID PRIMARY KEY,
  name TEXT,
  home FROZEN<address>
);
```

Collections and user-defined types should remain reasonably small. Large or unbounded collections can create oversized partitions and expensive updates.

---

# 6. Getting Started with CQL

`cqlsh` is Cassandra's interactive command-line shell.

```bash
cqlsh
cqlsh 192.168.1.10 9042
cqlsh -u username -p password
```

Useful shell commands:

```sql
DESCRIBE KEYSPACES;
DESCRIBE TABLES;
DESCRIBE TABLE users;
SELECT * FROM system.local;
```

## Creating a Blog Table

```sql
CREATE KEYSPACE blog
WITH replication = {
  'class': 'SimpleStrategy',
  'replication_factor': 1
};

USE blog;

CREATE TABLE posts_by_author (
  author_id UUID,
  created_at TIMESTAMP,
  post_id UUID,
  title TEXT,
  body TEXT,
  tags SET<TEXT>,
  PRIMARY KEY ((author_id), created_at, post_id)
) WITH CLUSTERING ORDER BY (created_at DESC);
```

## Writing and Reading Data

```sql
INSERT INTO posts_by_author (
  author_id,
  created_at,
  post_id,
  title,
  body
) VALUES (
  11111111-1111-1111-1111-111111111111,
  toTimestamp(now()),
  uuid(),
  'Hello Cassandra',
  'This is my first Cassandra post.'
);
```

Queries normally include the partition key:

```sql
SELECT *
FROM posts_by_author
WHERE author_id = 11111111-1111-1111-1111-111111111111;
```

Updates and deletes must identify the appropriate primary-key values:

```sql
UPDATE posts_by_author
SET title = 'Updated Title'
WHERE author_id = <author-id>
  AND created_at = <timestamp>
  AND post_id = <post-id>;

DELETE FROM posts_by_author
WHERE author_id = <author-id>
  AND created_at = <timestamp>
  AND post_id = <post-id>;
```

---

# 7. Query-Driven Data Modeling

Cassandra modeling requires a major shift from relational design.

| Relational Approach | Cassandra Approach |
|---|---|
| Normalize data | Denormalize data |
| Design around entities | Design around queries |
| Use joins | Store query-ready tables |
| Support ad hoc queries | Define access patterns first |
| Enforce foreign keys | Manage relationships in the application |

## The Golden Rule

> Know the queries first, then design a table for each query.

Suppose an application needs:

1. Get a user by email.
2. Get a user by user ID.

Cassandra may use two tables containing duplicated data:

```sql
CREATE TABLE users_by_email (
  email TEXT PRIMARY KEY,
  user_id UUID,
  username TEXT
);

CREATE TABLE users_by_id (
  user_id UUID PRIMARY KEY,
  email TEXT,
  username TEXT
);
```

This duplication is intentional. Each table directly supports one query without a join.

## Choosing a Partition Key

A good partition key:

- appears in the table's main queries
- has many distinct values
- distributes workload evenly
- avoids extremely large partitions

Poor partition key:

```sql
PRIMARY KEY (status)
```

If `status` only contains `active` and `inactive`, most data goes into two huge partitions.

Better partition key:

```sql
PRIMARY KEY (user_id)
```

Millions of user IDs distribute data more evenly.

## Clustering Columns

Clustering columns sort rows within a partition and support efficient range queries.

```sql
CREATE TABLE sensor_readings (
  sensor_id UUID,
  recorded_at TIMESTAMP,
  value DOUBLE,
  PRIMARY KEY ((sensor_id), recorded_at)
) WITH CLUSTERING ORDER BY (recorded_at DESC);
```

This table efficiently retrieves readings for one sensor over a time range.

## Preventing Large Partitions

Time-series workloads can continuously grow one partition. Add a time bucket to limit its size:

```sql
CREATE TABLE sensor_readings_by_month (
  sensor_id UUID,
  year_month TEXT,
  recorded_at TIMESTAMP,
  value DOUBLE,
  PRIMARY KEY ((sensor_id, year_month), recorded_at)
) WITH CLUSTERING ORDER BY (recorded_at DESC);
```

Now each sensor receives a new partition every month.

Partition-size limits depend on workload and Cassandra version, but smaller, predictable partitions are easier to operate than unbounded ones.

## Avoid `ALLOW FILTERING`

`ALLOW FILTERING` can make an otherwise rejected query run by scanning large amounts of data.

```sql
SELECT *
FROM users
WHERE city = 'Thimphu'
ALLOW FILTERING;
```

It may appear convenient during development but can become dangerously slow at scale. Create a query-specific table instead.

---

# 8. Consistency Levels

A consistency level specifies how many replicas must respond to a read or write.

| Consistency Level | Requirement |
|---|---|
| `ONE` | One replica responds |
| `TWO` | Two replicas respond |
| `THREE` | Three replicas respond |
| `QUORUM` | A majority of replicas responds |
| `LOCAL_QUORUM` | A majority in the local data center responds |
| `EACH_QUORUM` | A majority in every data center responds |
| `ALL` | Every replica responds |
| `ANY` | At least one node accepts a write or hint |

With replication factor `3`, a quorum is `2`.

A commonly used rule for strong read-after-write consistency is:

```text
read consistency level + write consistency level > replication factor
```

For example:

```text
QUORUM read + QUORUM write > RF 3
```

Higher consistency levels improve confidence that clients see recent data, but operations fail more easily when replicas are unavailable.

For multi-data-center applications, `LOCAL_QUORUM` often provides a useful balance by requiring a local majority without waiting for remote regions.

---

# 9. Lightweight Transactions

Normal Cassandra writes prioritize speed and availability. When an application needs compare-and-set behavior, Cassandra offers **lightweight transactions** using consensus.

Insert only if a user does not exist:

```sql
INSERT INTO users (user_id, email)
VALUES (uuid(), 'alice@example.com')
IF NOT EXISTS;
```

Update only if the existing value matches:

```sql
UPDATE users
SET email = 'new@example.com'
WHERE user_id = <user-id>
IF email = 'old@example.com';
```

Lightweight transactions require additional coordination and are significantly slower than normal writes. Use them only when the application truly requires conditional atomicity.

---

# 10. Cassandra Write and Read Paths

Understanding Cassandra's internal paths explains why it handles write-heavy workloads well.

## Write Path

```text
Write request
├──> Commit log on disk
└──> Memtable in memory
         |
         └── flushes to immutable SSTable on disk
```

1. The **commit log** records the write for crash recovery.
2. The **memtable** stores the write in memory.
3. When the memtable fills, Cassandra flushes it to an immutable **SSTable**.

Writes avoid modifying existing on-disk rows in place, making disk activity mostly sequential and efficient.

## Read Path

A read may need to combine information from the current memtable and several SSTables.

```text
Read request
├──> Caches
├──> Bloom filters
├──> Partition indexes
└──> Relevant SSTables
```

A **Bloom filter** can quickly determine that an SSTable definitely does not contain a requested partition, allowing Cassandra to skip unnecessary disk reads.

Bloom filters may produce false positives, but they do not produce false negatives.

---

# 11. Failure Recovery and Background Processes

## Hinted Handoff

If a replica is temporarily unavailable, the coordinator may store a **hint** containing the missed write.

When the replica returns, Cassandra replays the hint so the replica can catch up. Hinted handoff handles temporary outages but does not replace regular repair.

## Anti-Entropy Repair

Replicas can become inconsistent after outages or missed updates. Repair compares replicas and synchronizes differences.

```bash
nodetool repair
nodetool repair ecommerce
```

Cassandra uses Merkle trees to identify differing data efficiently instead of transferring entire datasets.

Repair must be planned and monitored as a regular operational process.

## Compaction

SSTables are immutable, so updates and deletes create newer versions instead of changing old files directly. Compaction merges SSTables, removes overwritten values, and eventually clears deletion markers called **tombstones**.

| Strategy | Suitable Workload |
|---|---|
| Size-Tiered Compaction Strategy | Write-heavy workloads |
| Leveled Compaction Strategy | Read-heavy workloads |
| Time-Window Compaction Strategy | Time-series and expiring data |

Configure a time-series table:

```sql
ALTER TABLE sensor_readings_by_month
WITH compaction = {
  'class': 'TimeWindowCompactionStrategy',
  'compaction_window_unit': 'HOURS',
  'compaction_window_size': 1
};
```

Too many tombstones can make reads slow. Applications should avoid delete-heavy designs and monitor tombstone behavior.

---

# 12. Operations and Monitoring

`nodetool` is Cassandra's main administrative command-line tool.

```bash
nodetool status
nodetool info
nodetool ring
nodetool repair
nodetool compactionstats
nodetool tablestats
```

Important configuration files include:

| File | Purpose |
|---|---|
| `cassandra.yaml` | Cluster, network, storage, and seed configuration |
| `cassandra-env.sh` | Environment and JVM settings |
| `jvm.options` | JVM tuning |
| `logback.xml` | Logging configuration |

Important `cassandra.yaml` settings include:

```yaml
cluster_name: "MyCluster"
listen_address: 192.168.1.12
native_transport_port: 9042
data_file_directories:
  - /var/lib/cassandra/data
commitlog_directory: /var/lib/cassandra/commitlog
```

Cassandra also maintains internal system keyspaces containing schema, authentication, topology, and repair metadata. These should never be manually deleted or modified.

---

# Quick Reference

## Core Concepts

| Concept | Meaning |
|---|---|
| Node | One Cassandra server |
| Data center | Logical or physical group of nodes |
| Keyspace | Top-level container with replication settings |
| Partition key | Determines data placement |
| Clustering column | Sorts rows inside a partition |
| Replication factor | Number of stored copies |
| Consistency level | Required replica responses |
| SSTable | Immutable sorted data file |
| Tombstone | Marker representing deleted data |

## Essential Commands

| Task | Command |
|---|---|
| Connect to Cassandra | `cqlsh` |
| Show keyspaces | `DESCRIBE KEYSPACES;` |
| Show node health | `nodetool status` |
| Synchronize replicas | `nodetool repair` |
| Inspect table statistics | `nodetool tablestats` |
| Inspect compactions | `nodetool compactionstats` |

## Design Checklist

- List application queries before creating tables.
- Create query-specific, denormalized tables.
- Select partition keys with high cardinality and even distribution.
- Use clustering columns for ordering and range queries.
- Bucket time-series data to prevent unbounded partitions.
- Choose replication and consistency settings for the workload.
- Avoid joins, foreign keys, and arbitrary filtering.
- Treat repair, compaction, and monitoring as routine operations.

---

# Key Takeaways

- Cassandra is a distributed, peer-to-peer column-family database.
- It provides horizontal scaling, replication, and no single point of failure.
- Partition keys determine data placement; clustering columns determine order within partitions.
- Cassandra tables must be designed around known queries.
- Denormalization and duplicated data are normal and often necessary.
- Tunable consistency balances availability, latency, and freshness.
- Writes use the commit log and memtable before becoming immutable SSTables.
- Hinted handoff, repair, and compaction keep replicated data healthy over time.
- Cassandra excels at large, write-heavy, always-available workloads.

Cassandra rewards careful planning. When its schema, partitioning, consistency levels, and operations match the application's access patterns, it can provide dependable performance across very large distributed workloads.
