# The Solution

## 1. Vertical Scaling and Database Choices

### a) Vertical Scaling

Use better servers.

### b) Database Choices

- **Postgres:** write speed is lower, but read speed is good compared to Cassandra.
- **Cassandra:** write speed is good, but read speed is lower compared to Postgres.

Generally, a DB node with 8–16 cores, 32–64 GB RAM and NVMe SSD can handle:

#### Write

| Database | Speed | Why |
|---|---|---|
| Postgres | ~5k–15k/s (single-row commits) | A write updates a B-tree in place, plus every index, plus the write-ahead log. |
| Cassandra | ~10k–30k/s | An append to a commit log plus an in-memory table, flushed to disk later (an LSM tree). |

#### Read (by key)

| Database | Speed | Why |
|---|---|---|
| Postgres | ~20k–100k/s | One B-tree lookup, usually served from memory. |
| Cassandra | ~5k–15k/s | The row can be spread across several disk files that have to be checked and merged. |

## 2. Sharding and Partitioning

## 3. Handling Bursts with Queue and Load Shedding

## 4. Batching and Hierarchical Aggregation



        
