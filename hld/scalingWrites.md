# Scaling Writes

Notes on how to handle a high write load, from the simplest fix (a bigger machine) to splitting hot keys.

## Contents

1. [Vertical scaling and database choices](#1-vertical-scaling-and-database-choices)
2. [Sharding and partitioning](#2-sharding-and-partitioning)
3. [Handling bursts with queues and load shedding](#3-handling-bursts-with-queues-and-load-shedding)
4. [Batching and hierarchical aggregation](#4-batching-and-hierarchical-aggregation)
5. [Resharding](#5-resharding)
6. [Hot keys](#6-hot-keys)

---

## 1. Vertical Scaling and Database Choices

### a) Vertical scaling

Use better servers: more cores, more RAM, faster disks. It is the cheapest change to make, so try it first.

### b) Database choice

- **Postgres:** slower writes, faster reads by key.
- **Cassandra:** faster writes, slower reads by key.

Rough throughput for one node with 8–16 cores, 32–64 GB RAM and an NVMe SSD:

#### Writes

| Database | Throughput | Why |
|---|---|---|
| Postgres | ~5k–15k/s (single-row commits) | A write updates a B-tree in place, plus every index, plus the write-ahead log. |
| Cassandra | ~10k–30k/s | An append to a commit log plus an in-memory table, flushed to disk later (an LSM tree). |

#### Reads (by key)

| Database | Throughput | Why |
|---|---|---|
| Postgres | ~20k–100k/s | One B-tree lookup, usually served from memory. |
| Cassandra | ~5k–15k/s | The row can be spread across several disk files that have to be checked and merged. |

These are ballpark figures; real numbers depend heavily on row size, indexes and hardware.

### c) Tuning any database for writes

- **Turn off expensive features** such as foreign key constraints, complex triggers or full-text indexing during high-write periods.
- **Tune the write-ahead log.** Postgres, for example, can group several transactions into one disk flush.
- **Keep fewer indexes.** Every index is updated on every write, so fewer indexes mean faster writes and slower reads.

---

## 2. Sharding and Partitioning

### Horizontal sharding

Split rows across several databases.

- **Pick a good partition key** so load spreads evenly across shards.
- **Use consistent hashing** so that adding a database moves only a small part of the data.
- **Add virtual nodes** to spread data more evenly between physical nodes.

### Vertical partitioning

Split columns into different stores, each chosen for how that data is used. For a social media post:

| Data | Access pattern | Storage |
|---|---|---|
| Post content | Read-heavy, rarely changes | Regular database with B-tree indexes |
| Post metrics (likes, views) | Very frequent updates | In-memory store or dedicated counters |
| Post analytics | Append-only, queried over time | Time-series or column-oriented database |

---

## 3. Handling Bursts with Queues and Load Shedding

### Write queues

Put a queue (for example Kafka) in front of the database when bursts are short-lived. The queue absorbs the spike and the database works through it at its own pace. This does not help if the load stays high, since the queue just keeps growing.

### Load shedding

When overloaded, drop the writes that do not affect the core business.

> Example: a food delivery app like Lieferando receives a GPS update from every courier every few seconds. Under load, it can drop some of those updates; the next one arrives soon anyway. Orders and payments are never dropped.

---

## 4. Batching and Hierarchical Aggregation

### Batching

Group many writes into one. This saves per-write overhead such as network round trips, transaction setup and index updates, and most databases process a batch faster than the same writes one by one.

Where the batching happens matters for durability:

| Where | Risk |
|---|---|
| In a durable queue like Kafka | No data loss; events stay in the log until processed. |
| In application memory, without a queue | Events still waiting in the batch are lost if the process crashes. |
| In database memory | Writes still in RAM are lost if the database crashes. |

Combining events into one write (such as turning 50 likes into `+50`) only helps when there are many events for the same key.

### Hierarchical aggregation

When millions of users interact with one item (for example, a viral post):

- **On the write side:** route groups of users to intermediate aggregator nodes. Each aggregator combines its events and sends one smaller update onward, so the service and database receive far less traffic.
- **On the read side:** everyone sees the same view of the post. Instead of one server pushing it to every user, it pushes to a set of broadcast nodes, and each of those pushes to its own group of users.

---

## 5. Resharding

How to move to a new shard layout without downtime:

1. **Dual-write:** send every new write to both the old and the new shard cluster.
2. **Backfill:** in the background, copy existing data from the old shards to the new ones.
3. **Switch over:** once the copy is complete and verified, serve all reads and writes from the new shards.
4. **Retire** the old shards.

---

## 6. Hot Keys

A single key (for example, the like count of a viral tweet) can receive more writes than one shard can handle.

### Fixed split

Split every key into a fixed number `k` of sub-keys and spread writes across them. Reads sum all `k` sub-keys. Simple, but every read pays the cost even for keys that are never hot.

### Dynamic split

Split only the keys that are actually hot.

> Example: a tweet receiving 100,000 likes per second is split across 100 sub-keys, each handling about 1,000 likes per second.

- Writers keep local statistics and, when a key looks hot, start spreading its writes across the sub-keys.
- **Readers and writers must agree on which keys are split.** If writers spread a key across sub-keys but readers only read the original key, counts come out wrong.
- The simplest safe approach is for readers to always check all sub-keys of a key.

Most production systems use dynamic splitting.
