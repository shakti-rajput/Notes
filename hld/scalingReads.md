1 - Optimize within your database
  a) Indexing
  b) Hardware Upgrades
  c) Denormalizing Strategies
2 - Scale your database  Horizontally
  a) Read Replicas (Single Leader)
  b) Database Sharding
3 - Add External Caching Layer
  a)  Application-Level Caching
    i) Time-based expiration
    ii) Write-through Invalidation
    iii) Write behind invalidation
    iv) Tagged invalidation
    v) Versioned Keys
  b) CDN and Edge Caching

"What happens when your queries start taking longer as your dataset grows?"
Just add indexes on columns you query frequently.
if (data skew)
  addCache()
else:
  addReplicas()

"How do you handle millions of concurrent reads for the same cached data?"(HOT KEY)
The first solution is request coalescing - basically combining multiple requests for the same key into a single request.

When coalescing isn't enough for extreme loads, you need to distribute the load itself.
Cache key fanout spreads a single hot key across multiple cache entries. Instead of storing the celebrity's post under one key, you store identical copies under ten different keys.
The trade-off with fanout is memory usage and cache consistency. 

"What happens when multiple requests try to rebuild an expired cache entry simultaneously?"
It's like a DDOS attack(cache stampede)
One approach uses distributed locks to serialize rebuilds. Only the first request to notice the missing cache entry gets to rebuild it, while everyone else waits for that rebuild to complete.
If the rebuild fails or takes too long, thousands of requests timeout waiting. You need complex timeout handling and fallback logic, making this approach fragile under load.

A smarter approach uses probabilistic early refresh 

"How do you handle cache invalidation when data updates need to be immediately visible?"
A common naive approach is delete the cache entry after a write.
A better approach for entity-level data is cache versioning.
U update the entry in db update its version in the db and update the cache version in db that for that field new version is the updated value.
So next request comes checks the updated which version is the updated value finds like v43 then find out the value from the cache is missing so go to the db to fetch and update the value of feild 

