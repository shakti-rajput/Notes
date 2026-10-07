# Scaling Reads

## Strategy Overview

1. **Optimize within your database**
   - Indexing, Queries themselves(N+1) problem
   - Hardware upgrades
   - Denormalization strategies
2. **Scale your database horizontally**
   - Read replicas (single leader)
   - Database sharding
3. **Add an external caching layer**
   - Application-level caching
     - Time-based expiration (TTL)
     - Write-through invalidation
     - Write-behind invalidation
     - Tagged invalidation
     - Versioned keys
   - CDN and edge caching

---

## Common Deep-Dive Questions

### 1. What happens when your queries start taking longer as your dataset grows?

Start by adding indexes on the columns you query frequently.

If indexing is not enough, the next step depends on the access pattern:

```
if (data skew):      # a small set of keys gets most of the traffic
    addCache()
else:                # traffic is spread evenly across the data
    addReplicas()
```

### 2. How do you handle millions of concurrent reads for the same cached data? (Hot key)

**Request coalescing**

Combine multiple in-flight requests for the same key into a single request. Only one request hits the backend; the rest wait and share its result.

**Cache key fanout**

When coalescing is not enough for extreme load, distribute the load itself. Spread a single hot key across multiple cache entries: instead of storing the celebrity's post under one key, store identical copies under ten different keys (e.g. `post:123:0` to `post:123:9`) and have each reader pick one at random.

Trade-offs:
- Higher memory usage
- Harder cache consistency (every copy must be updated or invalidated)

### 3. What happens when multiple requests try to rebuild an expired cache entry simultaneously? (Cache stampede)

When a popular entry expires, every request misses at once and hits the database together. The effect is similar to a DDoS attack on your own database.

**Distributed locks**

Serialize the rebuild. Only the first request that notices the missing entry rebuilds it; everyone else waits for that rebuild to complete.

Downsides:
- If the rebuild fails or is slow, thousands of requests time out while waiting
- Needs complex timeout handling and fallback logic
- Fragile under load

**Probabilistic early refresh (smarter approach)**

Refresh the entry *before* it expires. Each request has a small chance of triggering a rebuild, and that chance grows as the entry gets closer to expiry. One request refreshes the entry in the background while the others keep getting the still-valid cached value, so there is never a moment where everyone misses together.

### 4. How do you handle cache invalidation when data updates need to be immediately visible?

**Naive approach: delete after write**

Delete the cache entry after writing to the database. Simple, but prone to race conditions (a concurrent reader can repopulate the cache with stale data) and to failed deletes.

**Better approach for entity-level data: cache versioning**

Write path:
1. Update the entry in the database
2. Increment its version number in the database
3. Update the version pointer for that entity (e.g. `user:123:version = 43`)

Read path:
1. Look up the current version for the entity (finds `v43`)
2. Read the cache using the versioned key (e.g. `user:123:v43`)
3. On a miss, fetch from the database and populate the cache under that versioned key

Old versions are never read again and simply expire via TTL, so there is no need to delete them explicitly and no window where stale data is served.
