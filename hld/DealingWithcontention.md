# Dealing with Contention

---

# Part 1: My Notes

You want to buy a ticket, and two or more people are competing with each other for it.

### Resolving the conflict
- The best way to resolve the conflict is to move the decision into the database using **conditional writes**.
- The next step is **locking**. There are two ways to lock: **pessimistic** or **optimistic**.
  - If we assume contention will be rare, we use an **optimistic lock**.
  - If contention will be high, we use a **pessimistic lock**.
- Two common failures with pessimistic locking(FOR UPDATE; can be used to have lock on data which we are trying to update):
  1. Locks should not be held for too long.
  2. You should always acquire locks in the same order.

### Checking one row, updating another
- The methods above only work when the check and the update are on the same row.
- If you check one row but update a different row, you need the right **isolation level**. This problem is called **write skew**.

### Transactions across different databases
- You can create a lock in **Redis with a TTL**, so it expires after a set number of seconds.
  - Use **NX** in Redis so the key is only set if it does not already exist.
  - This is harmless when buying a seat, because the purchase is still checked with a conditional write at the time of purchase.
- Another method is a **fencing token**: a number that only increases. Every time the lock is granted, the token value increases, and the database checks it.
- We can also store the lock in the database itself: **who owns it right now** and **a TTL for how long they own it**. This is a lot slower than Redis.

### Scenarios
- **Online auction systems:** Optimistic concurrency. Bid prices only go up, so there is no ABA risk.
- **Ticketmaster / event booking:** When users select seats, reserve them immediately with a 10-minute TTL.
- **Banking / payment systems:** When the money movement stays in a single database, use pessimistic locking or optimistic concurrency control. When it spans multiple databases, use a multi-step process.
- **Ride sharing dispatch:** Set the driver's status to `pending_request` when sending a request, and clean it up automatically if the driver doesn't respond within a minute.
- **Flash sale / inventory systems:** Use optimistic concurrency with a dedicated version column for inventory updates, combined with temporary "cart holds" using a distributed lock with a TTL.
- **Yelp / review systems:** Use a version number when updating the average rating.

### When not to overcomplicate
- **Low-contention scenarios**, where conflicts are rare (like product descriptions that only admins can edit), can use basic optimistic concurrency with retry logic.
- **Single-user operations**, like personal to-do lists or private documents, need no coordination.
- **Read-heavy workloads**, with mostly reads and occasional writes, can use simple optimistic concurrency to handle the rare write conflict.

### "What about performance when everyone wants the same resource (one row of data)?"
- Use a **queue**.

---

# Part 2: Summary of the Page

## 1. The Problem: Race Conditions

- **Example:** one seat is left and Alice and Bob both click "Buy Now" at the same moment.
- The naive approach reads the seat count, checks it is above 0 in the app, then writes the decremented value back.
- Both read "1 seat", both pass the check, both pay. The count ends at **-1** and two people own the same seat.
- **Root cause:** reading and writing are two separate steps. The other request slips into the gap between them and acts on a stale value.
- This is called a **lost update**, the classic failure of a read-modify-write cycle.
- More users and more servers make the window more likely to be hit.
- **The general fix** is **compare-and-set**: make the write conditional on the value not having changed since you read it. Every technique below is a form of this.

---

## 2. Conditional Writes (start here)

When the rule is a simple condition on the current data, the database can check and write in **one atomic statement**.

```sql
UPDATE concerts
SET available_seats = available_seats - 1
WHERE concert_id = 'weeknd_tour' AND available_seats > 0;
```

- The database makes concurrent updates to the same row take turns. The second one re-checks the condition, finds it false and updates **0 rows**. One seat, one sale.
- At stricter isolation levels, the loser gets a serialization error instead and the app retries. The outcome is the same.

### Pitfall 1: 0 rows updated is not an error
A real purchase is two writes: decrement the seat and insert the ticket, wrapped in a transaction. But an UPDATE matching 0 rows still **succeeds**, so an unconditional INSERT would create a ticket for a seat you never got.

- **Fix:** make the insert depend on the update, e.g. `UPDATE ... RETURNING` feeding an `INSERT ... SELECT`.
- **Or:** check the affected row count in the app and roll back if it's 0.

### Pitfall 2: guard the actual contended resource
A seat counter only answers "is any seat left?", not "is seat A15 free?". With 20 seats left, two buyers can both decrement successfully and both get A15.

- **Fix:** give each seat its own row and guard that row:
  ```sql
  UPDATE tickets SET status = 'sold', user_id = 'user123'
  WHERE seat_number = 'A15' AND status = 'available';
  ```
- The seat count becomes derived data, or a cache updated in the same transaction.

**Same idea in other stores:** DynamoDB ConditionExpression, Redis `SET NX`, Cassandra lightweight transactions, HTTP `If-Match`.

**Limit:** this only works when the rule can be written in a `WHERE` clause.

---

## 3. Pessimistic Locking

**When:** the app must read, decide in code, then write, and the decision can't be a `WHERE` condition.

- **Example:** a group of 4 wants adjacent seats. Finding 4 free seats in a row happens in app code.
- **How:** lock the rows you read, decide while they're held, write, then commit.
  ```sql
  BEGIN;
  SELECT seat_number FROM seats
  WHERE section = 'floor' AND status = 'available' FOR UPDATE;
  -- app picks A15–A18
  UPDATE seats SET status = 'sold' WHERE seat_number IN ('A15','A16','A17','A18');
  COMMIT;
  ```
- A second group running the same query waits, and then sees only the seats that are left.
- **Cost:** every transaction pays for the lock, even when there was no conflict. Locking a whole section to buy 4 seats serializes heavy traffic.
- Other uses: checking a balance against a daily limit, or checking stock across several items for a bundle.
- If the logic simplifies back to a single condition, drop the lock and use a conditional write.

### Common failure modes
- **Locking too much or for too long:** lock as few rows as possible and release them quickly. Never do slow work like a payment API call while holding a lock; do it before or after.
- **Deadlocks:** two transactions grabbing the same rows in opposite orders wait forever on each other. Always lock in a consistent order (e.g. sorted by ID), and treat deadlock errors as retryable.

---

## 4. Optimistic Concurrency Control (OCC)

**When:** the same read-decide-write, but collisions are rare.

- Don't lock. Let everyone proceed and **detect** the conflict at write time.
- Keep a value that changes on every write, usually a **version number**:
  ```sql
  UPDATE concerts
  SET available_seats = available_seats - 1, version = version + 1
  WHERE concert_id = 'weeknd_tour' AND version = 42;  -- the version you read
  ```
- 0 rows updated means someone else wrote first: re-read and retry, or tell the user it's sold out.
- **Same pitfall:** check the row count and roll back, or a follow-up INSERT still runs.
- **Same idea elsewhere:** HTTP ETags / `If-Match` (412 on conflict), etcd revision, DynamoDB version attribute.

### What can serve as the version
- **Dedicated counter:** the safe default.
- **Last-updated timestamp:** fine if every write updates it and the clock is precise enough that two writes never get the same value.
- **Business value** (e.g. seat count, high bid): only safe if it moves in **one direction**, because of the **ABA problem**: a value goes A → B → A and the check can't see that anything changed.

**Rule of thumb:** frequent collisions → lock up front, since retries cost more than locks. Rare collisions (most e-commerce) → optimistic, and accept the occasional retry.

---

## 5. Isolation Levels and Write Skew

**Write skew:** two transactions read overlapping data, each makes a decision that's valid on its own, and they write **different rows**. Together they break a rule.

- **Example:** rule "at least one engineer on call". Alice and Bob are both on call. Each sees the other still on call and steps down at the same time. Now nobody is on call.
- Conditional writes, locks and versions can't catch this: there's no shared row. The conflict is in what each one **read**.

**The four isolation levels** (options, not a ladder):

| Level | Behaviour |
|---|---|
| READ UNCOMMITTED | Sees other transactions' uncommitted changes (rarely used) |
| READ COMMITTED | Sees only committed changes (Postgres default) |
| REPEATABLE READ | The same data reads the same within a transaction (MySQL default) |
| SERIALIZABLE | Transactions behave as if run one after another |

- Only **SERIALIZABLE** catches write skew: the database aborts one transaction, the app retries, re-reads, and backs off.
- **Cost:** the most expensive option, because the database tracks reads and writes and aborted work must be redone. Use it only for conflicts that span rows.
- Mostly a relational database feature; most NoSQL stores don't offer true SERIALIZABLE.
- **Often cheaper:** turn the cross-row rule into a single row, e.g. an on-call count on the team row, or a `FOR UPDATE` lock on the team row before stepping down. Then the earlier tools work.

**Big lesson:** the database can only protect a conflict it can see. Make sure the contended thing exists as one row, key or item.

---

## 6. Distributed Locks

**When:** exclusive access must outlive a single transaction, across a wait, a call to another service, or several steps.

- **Example:** Ticketmaster holds your seat for 10 minutes while you pay. `FOR UPDATE` can't do this: it would tie up a connection for 10 minutes, and other servers can't see it.
- **Fix:** store the lock as **data** (who holds it and when it expires), as a **lease** that any server can read.

| Option | How | Pros | Cons |
|---|---|---|---|
| **Redis with TTL** | `SET key value NX EX 600` (NX is critical, or a second process could overwrite the lock) | Fast, simple, expires automatically, visible to every server | Not airtight: if the holder pauses past the TTL (GC pause, slow network), two clients briefly both think they hold it. Fine for a soft seat hold, not when a double-grant would corrupt data. Redis is a single point of failure. |
| **Database columns** | `UPDATE seats SET reserved_by = ?, reserved_until = NOW() + 10 min WHERE seat_id = ? AND (reserved_until IS NULL OR reserved_until < NOW())` | No new infrastructure; same consistency as your data; expired holds count as free, so no cleanup job is needed | Slower than a cache; the lock row can become a contention point |
| **ZooKeeper / etcd** | Ephemeral nodes or leases, backed by consensus (ZAB / Raft) | Strongest guarantees, even during network partitions; locks clean up automatically when a client dies | Running a separate coordination cluster is operational overhead |

- Reservations are a **UX win** too: the competition shrinks from the whole checkout to the moment of selection, so nobody fills in payment details only to lose the seat.

---

## 7. Keep It in One Database

- Every technique here depends on the contended resource living in **one authoritative store**. Nine times out of ten that's possible.
- If one operation must span several services or shards (e.g. a transfer between two sharded accounts), it's a **distributed transaction**, covered in Multi-step Processes.
- If the same record is writable in several places at once (multi-leader or leaderless replication), it's **conflict resolution**: last-write-wins, vector clocks or CRDTs.

---

## 8. Choosing the Right Approach

Walk down the list and take the first one that fits:

1. **The check is a condition on the row you write** → conditional write.
2. **Read, decide in code, write; high contention** → pessimistic locking.
3. **Same, but collisions are rare** → optimistic concurrency.
4. **The rule spans rows that never collide** → SERIALIZABLE, or move the rule onto one row.
5. **The hold must outlive one transaction** → distributed lock.

| Approach | Use when | Avoid when | Latency | Complexity |
|---|---|---|---|---|
| Conditional write | Check is a condition on the written row | Decision needs app logic or other rows | Low | Low |
| Pessimistic locking | Read-decide-write; high contention | Low contention, or a conditional write is enough | Low, but others wait | Low |
| Optimistic concurrency | Read-decide-write; rare collisions | High contention (retries pile up) | Low; retry cost on conflict | Medium |
| SERIALIZABLE | Write skew / rules across rows | Hot, high-contention paths | Medium | Medium |
| Distributed lock | Hold spans a wait, external call or several steps | A single-row guard in one transaction is enough | Low | Medium |

**When in doubt:** keep it in one database and use the simplest tool that fits.

---

## 9. In Interviews

**Recognition signals:** watch for these and raise contention yourself:
- many users competing for limited resources (tickets, auctions, flash sales, matching drivers to riders),
- risk of double-booking or double-charging,
- consistency under high concurrency (balances, inventory),
- the same operation running at once on several servers.

### Common scenarios
- **Online auction:** optimistic concurrency, using the current high bid as the version (bids only go up, so no ABA risk).
- **Ticketmaster:** temporary 10-minute seat reservations with a distributed lock. The bigger win over plain locking is the user experience.
- **Banking / payments:** within one database, pessimistic locking or OCC. Across services or shards, it's a distributed transaction (Multi-step Processes).
- **Ride sharing:** set the driver to `pending_request`, with a TTL in a cache or an expiry column so an unanswered request frees up automatically (about 10 seconds).
- **Flash sale / inventory:** OCC with a version column, plus temporary cart holds using a distributed lock with a TTL.
- **Yelp / reviews:** OCC with a version column when updating the average rating.

**Strong candidates say it up front**, e.g. "Bidders compete for items, so I'll use optimistic concurrency with the high bid as the version."

### When not to overcomplicate
- Don't reach for Redis locks when a database transaction with row locks or OCC is enough. Every new component adds failure modes.
- **Low contention** (admin-only edits): basic OCC with retries.
- **Single-user data** (to-do lists, private documents, preferences): no coordination needed.
- **Read-heavy workloads**: simple OCC for the rare write conflict.

---

## 10. Common Deep Dives

### "How do you prevent deadlocks?"
- **Example:** Alice sends money to Bob while Bob sends money to Alice. Transaction 1 locks Alice then waits for Bob; transaction 2 locks Bob then waits for Alice. Both wait forever.
- **Fix: ordered locking.** Always lock in a global order, e.g. by sorted user ID, regardless of who started the transfer. Not "lock the initiator first".
- Databases detect deadlocks automatically and abort one transaction. Catch that error and retry. Lock-wait timeouts are a coarser backstop for locks held too long.

### "How do you handle the ABA problem?"
- **Example:** using `review_count` as the version. You read 100, a review is deleted (99), a new one is added (100). Your check passes and silently overwrites the change in between.
- **Fix:** a dedicated version column that increments on every update.
- **Fallback:** put every field you read into the `WHERE` clause, or use a built-in row version (e.g. Postgres `xmin`).

### "What if everyone wants the same resource?"
- The hot partition / celebrity problem: a celebrity's follower count, a rare eBay item, a surprise Taylor Swift concert.
- Normal scaling doesn't help: sharding can't split one row, load balancing just queues on that row, read replicas don't help writes.
- **First, change the problem:** split into several identical items, or accept eventual consistency (e.g. follower counts can lag a few seconds).
- **If strong consistency is required:** queue-based serialization. All requests for that resource go into one queue processed by **one worker**, so there's no contention. The queue also absorbs spikes.
- **Trade-off:** throughput is capped at what one worker can handle, and the worker is a single point of failure, so keep a standby. Writes that need ordering must go to the same queue partition.

---

## Conclusion

- Every contended resource has **one source of truth**, and that's where correctness is enforced.
- Conditional writes, pessimistic locking, OCC and isolation levels are different ways to coordinate access there, escalating as the gap to protect grows.
- Distributed locks handle exclusivity across waits or other systems. They protect the source of truth; they don't replace it.
- A database like Postgres can handle more contention than most people assume. Add external locks only when traffic or user experience demands it.
- Once an operation spans several sources of truth, it's no longer contention but a distributed transaction.
- **Simplest working solution at the source of truth wins.**
