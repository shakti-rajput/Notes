The Solution:

1- Vertical Scaling and Database Choices
    a) Vertical Scaling - Use better Servers
    b) Database Choices - 
      Postgress DB - Write speed is less. But read speed if good compared to Cassandra.
      Cassandra write speed is good but read speed is less compared to postgress.
      Generally - 8–16 cores, 32–64 GB RAM and NVMe SSD DB can handle
      Write - 
      Postgers - ~5k–15k/s (single-row commits) [Write]  - write updates a B-tree in place, plus every index, plus the write-ahead log
      Cassandra -  ~10k–30k/s [Write] - an append to a commit log plus an in-memory table, flushed to disk later (an LSM tree).
      Read - Reads by key
      Postgres - 	~20k–100k/s - One B-tree lookup, usually served from memory.
      Cassandra - ~5k–15k/s - The row can be anywhere on disk files that have to be checked and merged. 
      How cassandra reads the data- 
        1 - In-memory table:
        2 - Bloom Filter (per SSTable) - Most files are ruled out here without touching the disk.
        3 - Index (Per SSTable) - an index gives the position of key 42 in the file.
        4 - It just Read that row from that position
      
2- Sharding and Partitioning
3- Handling Bursts with Queue and Load Shedding
4- Batching and Hierarical Aggregation
