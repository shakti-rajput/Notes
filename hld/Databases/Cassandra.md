# Cassandra

## Write

1. The write request arrives at any one node, which becomes the **coordinator**.
2. The coordinator hashes the partition key into a token and finds the replicas for that key.
   - The coordinator may or may not be one of the key's replicas.
3. The coordinator sends the write to all the replicas in parallel.
4. Each replica appends the write to its commit log, inserts it into its memtable, and acks the coordinator.
5. **Ack to the client** once enough replicas have acked for the consistency level (with quorum and 3 replicas, that is 2).

**Quorum:** the number of replicas that must confirm the write before it counts as successful. It is a majority of the key's replicas (the replication factor), not of all nodes in the cluster: `quorum = (replication factor / 2) + 1`, rounded down. With replication factor 3, quorum is 2. The coordinator only counts if it is itself a replica for the key.

### Later, in the background

- The remaining replicas (beyond the quorum number) finish the write.
- Once a memtable fills up, it is flushed to an SSTable in one sequential pass.
- Compaction merges SSTables.

### SSTable facts

- An SSTable is immutable.
- It is sorted by partition key (token), which is what makes it fast to find a key inside an SSTable.
- Finding the right SSTable when reading is the bloom filter's job.

## Read

1. The read request reaches the coordinator, which hashes the partition key into a token.
2. The coordinator finds the replicas and asks as many as the consistency level requires.
3. Each replica checks its memtable (in-memory data that has not been flushed to SSTables yet).
4. Each replica checks the bloom filter (`Filter.db`) of every SSTable of that table, and skips the SSTables that definitely do not contain the key.
5. For the remaining SSTables, it uses the index (`Summary.db`, then `Index.db`) to get the byte position of the partition in `Data.db`, jumps to that position, and reads only those rows.
6. The replica merges all versions found, newest timestamp winning.
7. **Response to the client:** the coordinator compares the replica answers and returns the newest.

### Files in an SSTable

| File | What it holds |
|---|---|
| `Data.db` | The actual rows: partitions, rows, cell values, timestamps, tombstones |
| `Index.db` | For each partition key, its byte position in `Data.db` |
| `Summary.db` | A sample of the index (for example every 128th key), kept in memory |
| `Filter.db` | The bloom filter, kept in memory |
