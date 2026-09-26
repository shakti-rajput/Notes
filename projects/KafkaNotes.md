# Event-Driven E-Commerce Platform — Interview Prep Guide

A complete guide to explaining, defending, and being grilled on the shopping saga project: architecture, every table, every Kafka message (who sends it, who reads it, why), failure scenarios, and a question bank with answers.

---

## 0. Review of your existing write-up — what's wrong with it

The concepts in it are mostly correct. The problem is that it describes a slightly different system than the one you designed and are building, and an interviewer who asks one follow-up question will find the gaps.

| # | Issue | Why it matters | Fix |
|---|---|---|---|
| 1 | Mentions a **Shipping Service** | It doesn't exist. Your six services are catalog, cart, order, inventory, payment, analytics. | Remove shipping from the saga, or say "shipping would be the next saga step" if asked. |
| 2 | "Why event-driven" diagram shows **Order Service publishing `OrderCreated` and everyone consuming it in parallel** | That's **choreography**. Your design is **orchestration**: order-service sends one command at a time, in sequence. In the parallel version, payment could charge the customer before inventory knows whether there's stock. If you draw that diagram and then describe an orchestrator, the interviewer will spot the contradiction. | Use the sequential command/result diagram in section 5. |
| 3 | Exactly-once section doesn't mention **`isolation.level=read_committed`** | Transactional producers only help if consumers skip aborted records. With the default (`read_uncommitted`), a consumer can read a message from a transaction that later aborted. This is the most common follow-up question on EOS. | Add it (see 7.3). |
| 4 | `producer.sendOffsetsToTransaction(offsets, consumerGroupId)` | The `String groupId` overload is deprecated; the current API takes `consumer.groupMetadata()`. Also, in your Spring code you never call this yourself: the listener container does it when a `KafkaTransactionManager` is configured. | Say "Spring Kafka's transactional listener container sends the offsets inside the transaction for me." |
| 5 | Missing the **Transactional Outbox** | It's the most distinctive design decision in the project, and it's the answer to the doc's own "honest caveat" (Kafka transactions can't span your database). | See 7.2. |
| 6 | Says "**deployed it on Kubernetes with Prometheus and Grafana**", dated **2025** | Today, catalog-service runs locally with Postgres and Elasticsearch. Interviewers often ask "can you show me?" or "what broke when you deployed it?". | Describe what's running as running and the rest as designed/in progress, and update the claims as each step is actually done (section 10). |

---

## 1. The pitch

### 30 seconds
"I built an e-commerce backend as six Spring Boot services that coordinate checkout through Kafka instead of direct calls. Checkout is a distributed transaction: reserve stock, then charge payment, each in a different service with its own database. I implemented it as an orchestrated saga: if payment is declined, the orchestrator issues a compensating command that releases the reserved stock. The interesting engineering is in making that reliable: a transactional outbox so a database change and the Kafka message it triggers can't get out of sync, Kafka exactly-once transactions in the services that consume and produce, and idempotent consumers everywhere, because in a real system messages do get delivered twice."

### 2 minutes (add these after the 30-second version)
1. **Data**: "Each service owns a store chosen for its job: Postgres for orders, stock and payments (needs transactions and conditional updates), Elasticsearch for product search, Redis for carts (ephemeral), MongoDB for analytics events (write-once documents queried by aggregation)."
2. **Analytics**: "A separate analytics service consumes order outcomes. It runs a Kafka Streams topology for live windowed counts and stores raw events in MongoDB for revenue reports."
3. **Ops**: "Everything runs in Docker Compose locally, with Kubernetes manifests and Prometheus/Grafana, where consumer lag is the key metric."
4. **Why**: "I deliberately split it into services to work through the real problems: partial failure, duplicates, eventual consistency. For a product this size, a monolith would be the honest choice."

---

## 2. Architecture

```
                           ┌──────────────┐
                           │   Frontend   │  React
                           └──────┬───────┘
                 HTTP             │             HTTP
        ┌─────────────────────────┼─────────────────────────┐
        ▼                         ▼                         ▼
┌───────────────┐        ┌───────────────┐         ┌────────────────┐
│catalog-service│◀──HTTP─│ cart-service  │         │analytics-service│
│Postgres + ES  │  price │    Redis      │         │ Kafka Streams + │
└──────┬────────┘        └──────┬────────┘         │    MongoDB      │
       │ HTTP (stock)           │ HTTP POST        └────────▲────────┘
       │                        │ /api/orders               │
       ▼                        ▼                           │ order.confirmed
┌───────────────┐  HTTP  ┌───────────────┐                  │ order.failed
│inventory-svc  │◀───────│ order-service │──────────────────┘ cart.item.added
│Postgres       │ (price │ ORCHESTRATOR  │                     (from cart)
└──────▲──┬─────┘ lookup │Postgres+Outbox│
       │  │   to catalog)└───┬───────▲───┘
       │  │                  │       │
       │  └── inventory.reserve.result / inventory.release.result ──┘
       └───── inventory.reserve.requested / inventory.release.requested
                             │       ▲
                             ▼       │
                      payment.charge.requested / payment.charge.result
                             │       │
                        ┌────▼───────┴──┐
                        │payment-service│
                        │   Postgres    │
                        └───────────────┘
```

Rule of thumb for the design:
- **Reads** that other services need right now → **synchronous HTTP** (price lookup, stock display). Nothing to compensate, so no reason to go through Kafka.
- **State changes that are part of the checkout** → **Kafka commands and results**, coordinated by order-service.
- **Facts other services might care about** (order finished, item added to cart) → **Kafka events**, fire and forget.

### Services at a glance

| Service | Port | Store(s) | Responsibility | Kafka role |
|---|---|---|---|---|
| catalog-service | 8080 (8081 on your machine) | Postgres + Elasticsearch | Products, search | none |
| cart-service | 8083 | Redis | Cart, hands off checkout | produces `cart.item.added` |
| order-service | 8082 | Postgres | Saga orchestrator, order state | produces commands + outcomes, consumes results |
| inventory-service | 8090 | Postgres | Reserve / release stock | consumes commands, produces results (transactional) |
| payment-service | 8091 | Postgres | Charge | consumes commands, produces results (transactional) |
| analytics-service | 8085 | MongoDB + Streams state store | Reporting | consumes outcomes + cart events |

---

## 3. Data model — every table, index, and key

### catalog-service (Postgres `catalog` + Elasticsearch)

**Table `products`** (this is exactly what Hibernate created in your log)

| Column | Type | Notes |
|---|---|---|
| id | varchar(255) PK | UUID, generated in Java |
| name | varchar(255) | |
| description | varchar(255) | |
| category | varchar(255) | |
| price | numeric(38,2) | `BigDecimal` in Java: exact money arithmetic |

**Elasticsearch index `products`** (search copy, derived from the table)

| Field | ES type | Why |
|---|---|---|
| id | keyword (the `@Id`) | same id as the Postgres row |
| name | text | analyzed → full-text search |
| description | text | analyzed → full-text search |
| category | keyword | exact match / filtering |
| price | double | sort / range queries |

### cart-service (Redis)

| Key | Type | Content |
|---|---|---|
| `cart:{userId}` | hash | field = `productId`, value = quantity |

`HINCRBY` makes "add 1 more" atomic. No table, no schema: carts are disposable.

### order-service (Postgres `orders`)

**`orders`**

| Column | Type | Notes |
|---|---|---|
| id | varchar PK | UUID, also the Kafka message key |
| user_id | varchar | |
| status | varchar | `PENDING_RESERVATION`, `CHARGING`, `RELEASING`, `CONFIRMED`, `FAILED` |
| total_amount | numeric | computed from **catalog prices**, never trusted from the client |
| failure_reason | varchar | |
| created_at / updated_at | timestamp | |

**`order_items`**

| Column | Type | Notes |
|---|---|---|
| id | bigserial PK | |
| order_id | FK → orders.id | |
| product_id, product_name | varchar | name copied in: snapshot at purchase time |
| quantity | int | |
| unit_price | numeric | price snapshot at purchase time |

**`outbox_events`** (the Transactional Outbox)

| Column | Type | Notes |
|---|---|---|
| id | varchar PK | UUID; becomes the `eventId` inside the payload = downstream idempotency key |
| topic | varchar | where the poller should publish it |
| message_key | varchar | the orderId → decides the Kafka partition |
| payload | text | JSON, serialized once at insert time |
| published | boolean | false until the poller has sent it |
| created_at | timestamp | poller publishes oldest first |

### inventory-service (Postgres `inventory`)

**`inventory_items`**

| Column | Type | Notes |
|---|---|---|
| product_id | varchar PK | same id as catalog |
| stock | int | reserve = `UPDATE ... SET stock = stock - :qty WHERE product_id = :id AND stock >= :qty` |
| version | bigint | JPA `@Version`, optimistic locking |

**`processed_events`** (idempotency ledger)

| Column | Type | Notes |
|---|---|---|
| event_id | varchar PK | the command's eventId; PK makes a duplicate insert fail |
| outcome | varchar | RESERVED / RELEASED, for debugging |
| processed_at | timestamp | |

### payment-service (Postgres `payments`)

**`payments`**

| Column | Type | Notes |
|---|---|---|
| event_id | varchar PK | the charge command's eventId: the PK **is** the idempotency guard |
| order_id | varchar | |
| user_id | varchar | |
| amount | numeric | |
| status | varchar | `CHARGED` / `DECLINED` |
| reason | varchar | |
| created_at | timestamp | |

### analytics-service (MongoDB `analytics` + Kafka Streams)

**Collection `order_events`**
```json
{
  "_id": "3f2a…:CONFIRMED",
  "orderId": "3f2a…",
  "userId": "demo-user",
  "status": "CONFIRMED",
  "reason": null,
  "totalAmount": 37.00,
  "items": [ { "productId": "p1", "productName": "Test Mug", "quantity": 2, "unitPrice": 12.50 } ],
  "recordedAt": "2026-08-24T10:00:00Z"
}
```
`_id = orderId:status` means a redelivered event **overwrites** rather than duplicates: idempotent by construction.

**Collection `cart_events`**: `{ userId, productId, quantity, timestamp }`

**Kafka Streams state store `order-status-counts`**: windowed store, key = status, value = count per 1-minute window. Lives in RocksDB on the analytics pod, backed up to a Kafka changelog topic.

### Why one database per service
Each service can change its schema, scale, and fail independently. The cost: no joins and no transactions across services. That's why the saga exists.

---

## 4. The Kafka contract — who publishes what, to whom, and why

### Commands vs events (say this out loud in interviews)
- **Command**: "please do X", addressed to exactly one service, which must answer. Named `<service>.<action>.requested`.
- **Result**: the answer to a command. Named `<service>.<action>.result`.
- **Event**: "X happened", a fact. The sender doesn't know or care who listens. `order.confirmed`, `cart.item.added`.

### All topics

| Topic | Producer | Consumer(s) | Key | Type | Why it exists |
|---|---|---|---|---|---|
| `inventory.reserve.requested` | order-service (via outbox) | inventory-service | orderId | command | Hold stock for a new order |
| `inventory.reserve.result` | inventory-service | order-service | orderId | result | Tell the orchestrator whether stock was held |
| `payment.charge.requested` | order-service (via outbox) | payment-service | orderId | command | Charge the customer once stock is held |
| `payment.charge.result` | payment-service | order-service | orderId | result | Tell the orchestrator whether the charge worked |
| `inventory.release.requested` | order-service (via outbox) | inventory-service | orderId | command | **Compensation**: give the stock back after a declined payment |
| `inventory.release.result` | inventory-service | order-service | orderId | result | Confirm compensation finished → order can become FAILED |
| `order.confirmed` | order-service (via outbox) | analytics-service | orderId | event | Order finished successfully |
| `order.failed` | order-service (via outbox) | analytics-service | orderId | event | Order finished unsuccessfully |
| `cart.item.added` | cart-service (direct, fire-and-forget) | analytics-service | userId | event | Funnel analytics; allowed to be lossy |

All topics: 3 partitions (except anything you add for DLQ), replication factor 1 locally (3 in production).

> **Bug to avoid when you build inventory-service**: in the earlier generated zip, the release result was published to `inventory.reserve.result` instead of `inventory.release.result`. order-service would ignore it (the order is in `RELEASING`, not `PENDING_RESERVATION`) and the order would stay stuck in `RELEASING` forever. Publish release results to their own topic.

### Example payloads

`inventory.reserve.requested` (also the shape of `inventory.release.requested`):
```json
{ "orderId": "3f2a…", "eventId": "a91c…", "items": [ { "productId": "p1", "quantity": 2 } ] }
```

`inventory.reserve.result`:
```json
{ "orderId": "3f2a…", "eventId": "a91c…", "status": "RESERVED", "reason": null }
{ "orderId": "3f2a…", "eventId": "a91c…", "status": "INSUFFICIENT_STOCK", "reason": "One or more items are out of stock" }
```

`payment.charge.requested`:
```json
{ "orderId": "3f2a…", "eventId": "c07d…", "userId": "demo-user", "amount": 37.00 }
```

`payment.charge.result`:
```json
{ "orderId": "3f2a…", "eventId": "c07d…", "status": "DECLINED", "reason": "Card declined by simulated gateway" }
```

`order.confirmed` / `order.failed`: the full order (items + total), so analytics never has to call order-service back. This is called **event-carried state transfer**.

`cart.item.added`:
```json
{ "userId": "demo-user", "productId": "p1", "quantity": 1, "timestamp": "2026-08-24T10:00:00Z" }
```

### Why every saga message is keyed by `orderId`
Kafka only guarantees order **within a partition**. Same key → same partition → messages about one order are processed in the order they were sent, by one consumer at a time. Different orders spread across partitions and run in parallel.

### What `eventId` is for
Every command carries a unique `eventId` (the outbox row id). It's the same across redeliveries of the same message and different for every new command. Consumers record it (`processed_events`, `payments.event_id`) so a redelivered command is recognized and not applied twice.

---

## 5. Interactions, step by step

### 5.1 Browse and search (synchronous, no Kafka)
1. Frontend → `GET catalog/api/products/search?q=mug`
2. catalog queries the Elasticsearch `products` index (full-text, case-insensitive).
3. For stock display, catalog calls `GET inventory/api/inventory/{id}`. If inventory is down, it shows "unknown" rather than failing the page.

### 5.2 Add to cart
1. Frontend → `POST cart/api/cart/{userId}/items {productId, quantity}`
2. cart runs `HINCRBY cart:{userId} {productId} {qty}` in Redis.
3. cart publishes `cart.item.added` (no outbox, no transaction: losing one analytics event is acceptable).
4. `GET cart` resolves current prices from catalog over HTTP.

### 5.3 Checkout, happy path

| Step | Who | Database change | Kafka message |
|---|---|---|---|
| 1 | Frontend → cart `POST /checkout` | | |
| 2 | cart → order `POST /api/orders` (HTTP) | | |
| 3 | order fetches each price from catalog (HTTP) | | |
| 4 | order, **one DB transaction** | insert `orders` (PENDING_RESERVATION) + `order_items` + `outbox_events` row | |
| 5 | order returns 202 Accepted with the orderId; cart clears the Redis key | | |
| 6 | order's outbox poller (every 300 ms) | mark outbox row `published = true` | → `inventory.reserve.requested` |
| 7 | inventory, **inside a Kafka transaction** | conditional stock decrement + insert `processed_events` | → `inventory.reserve.result` = RESERVED, offset committed in the same Kafka transaction |
| 8 | order consumes the result, **one DB transaction** | order → CHARGING + new outbox row | |
| 9 | poller | outbox row published | → `payment.charge.requested` |
| 10 | payment, inside a Kafka transaction | insert `payments` (CHARGED) | → `payment.charge.result` = CHARGED |
| 11 | order consumes it | order → CONFIRMED + outbox row | |
| 12 | poller | | → `order.confirmed` |
| 13 | analytics | upsert into `order_events`; Streams increments the CONFIRMED count for this minute | |

The frontend polls `GET /api/orders/{id}` and sees the status move. The HTTP call returns immediately; the saga finishes asynchronously.

### 5.4 Out of stock
Steps 1–6 as above. At step 7, the conditional update matches 0 rows for some item:
- inventory marks its **DB** transaction rollback-only (so any items it already decremented in this command are undone: all or nothing) but **returns normally**, so the **Kafka** transaction still commits and publishes `INSUFFICIENT_STOCK`.
- order → FAILED + `order.failed` via outbox.
- No compensation needed: nothing was reserved.

Why not throw an exception? An exception aborts the Kafka transaction, the offset isn't committed, and the same command is redelivered forever. "Out of stock" isn't a transient error; retrying won't fix it.

### 5.5 Payment declined → compensation

| Step | Who | Change | Message |
|---|---|---|---|
| 10' | payment | insert `payments` (DECLINED) | → `payment.charge.result` = DECLINED |
| 11' | order | order → RELEASING, failure_reason set, outbox row | |
| 12' | poller | | → `inventory.release.requested` (same items) |
| 13' | inventory | `stock = stock + qty`, insert `processed_events` | → `inventory.release.result` = RELEASED |
| 14' | order | order → FAILED, outbox row | → `order.failed` |

Payment does **not** retry a declined charge (risk of double charge). Recovering from a payment failure is the orchestrator's job: it compensates.

### 5.6 Order state machine
```
PENDING_RESERVATION ──RESERVED──▶ CHARGING ──CHARGED──▶ CONFIRMED
        │                            │
 INSUFFICIENT_STOCK               DECLINED
        │                            │
        ▼                            ▼
      FAILED ◀───────RELEASED─── RELEASING
```
Every handler checks the current state first ("only handle a reserve result if the order is PENDING_RESERVATION"). A late or duplicate result for a state the order has already left is ignored. That's the orchestrator's own idempotency.

---

## 6. Failure scenarios — "what happens if X crashes right here?"

This is where interviews are won or lost. Practice drawing each one.

| # | Failure | What happens | What protects you |
|---|---|---|---|
| 1 | order-service crashes **after** committing order + outbox row, **before** publishing | On restart the poller finds `published = false` and sends it | Outbox: state and "message to send" committed atomically |
| 2 | Poller sends to Kafka, crashes **before** marking the row published | On restart it sends the same message again (same eventId) | Idempotent consumers: inventory sees the eventId in `processed_events` and doesn't reserve twice |
| 3 | order-service crashes **before** committing | Nothing was written, nothing was sent; client gets an error and retries | Plain DB transaction |
| 4 | inventory crashes mid-processing, before the Kafka transaction commits | Kafka transaction aborts, offset not committed, command redelivered | Kafka EOS + read_committed |
| 5 | inventory's **DB** commits, then it crashes before the **Kafka** transaction commits | Command redelivered, but stock was already decremented | `processed_events` says ALREADY_PROCESSED → republish RESERVED without decrementing again. **This is why you need the ledger even with Kafka EOS**: the DB and Kafka transactions are two separate commits |
| 6 | Same result delivered twice to order-service | Second copy arrives when the order has already moved on | State guard in every handler |
| 7 | payment-service is down for 10 minutes | Commands pile up in `payment.charge.requested`; orders sit in CHARGING; consumer lag grows | Kafka retains the messages; payment catches up on restart; the lag alert tells you |
| 8 | Kafka broker down | Outbox rows accumulate as unpublished; checkouts are still accepted | Outbox decouples accepting an order from Kafka being available |
| 9 | Compensation (release) keeps failing | Order stuck in RELEASING | Release is idempotent so retrying is safe; after N attempts → DLQ + alert, human reconciles. There's no fully automatic answer |
| 10 | inventory never answers at all | Order stuck in PENDING_RESERVATION | **Not handled yet**: needs a saga timeout job (see 9) |
| 11 | A malformed "poison" message | Consumer throws on every retry, partition blocked behind it | Spring Kafka `DefaultErrorHandler` with backoff + `DeadLetterPublishingRecoverer` → move it to `<topic>.DLT` so the partition keeps moving |
| 12 | Two orders race for the last unit | Both issue `UPDATE ... WHERE stock >= 1`; the database serializes the row update; exactly one gets 1 row updated | Conditional update, no read-then-write gap |
| 13 | Payment gateway **times out** (we don't know if it charged) | Different from DECLINED: the charge may have happened | Send an idempotency key to the gateway and retry with the same key, or reconcile later against the gateway (the same idea as the overnight reconciliation job you built at OLA) |

---

## 7. Deep dives — things to explain on a whiteboard

### 7.1 Orchestration vs choreography — why orchestration here
- **Choreography**: each service reacts to events and emits new ones. No coordinator. Fine for 2–3 steps; beyond that the flow is spread across services and "what state is this order in?" has no single answer.
- **Orchestration**: order-service holds the state machine and tells each participant what to do. One place to read the flow, one table to query the status, one place to add a step.
- Trade-off you accept: order-service knows the sequence (some coupling) and is a critical component.
- Participants stay dumb and reusable: inventory doesn't know payment exists.

### 7.2 Transactional Outbox
**Problem (the dual-write problem):** order-service must (a) update Postgres and (b) send a Kafka message. Two systems, no shared transaction.
- Write DB, then send → crash in between → order stuck, message never sent.
- Send, then write DB → crash in between → inventory reserves stock for an order that doesn't exist.

**Solution:** insert the message into an `outbox_events` table **in the same DB transaction** as the state change. A poller publishes unpublished rows and marks them published.

**Result:** atomic "state changed + message queued", and **at-least-once** publishing (step 2 of section 6 can duplicate), which is why every consumer is idempotent.

**Alternatives to know:** Change Data Capture (Debezium reads the Postgres WAL and publishes outbox rows, no polling); 2PC/XA (correct in theory, rarely used: slow, operationally painful, Kafka doesn't support XA).

**Multiple order-service replicas:** the poller query uses a row lock; in Postgres you'd use `SELECT ... FOR UPDATE SKIP LOCKED` so replicas take different rows instead of queuing on the same ones.

### 7.3 Exactly-once in inventory and payment
Their job is **consume → process → produce**. Kafka transactions make these atomic:
- the result message,
- the consumer offset commit

...are committed together or not at all.

Pieces:
1. **Idempotent producer** (`enable.idempotence=true`, default since Kafka 3.0): producer id + per-partition sequence numbers; the broker drops duplicates from producer retries.
2. **Transactional producer** (`transaction-id-prefix` in Spring): gives a stable `transactional.id` so a restarted instance fences off its old "zombie" self.
3. **Offsets in the transaction**: Spring's listener container calls `sendOffsetsToTransaction` for you.
4. **`isolation.level=read_committed`** on every consumer downstream: otherwise consumers can read records from aborted transactions.

**The caveat, precisely:** exactly-once holds **inside Kafka**. The Postgres write is a separate commit. That's why inventory and payment still have their idempotency ledgers (scenario 5).

**Cost:** extra round trips and transaction markers → lower throughput, slightly higher latency. Worth it for money and stock, not for `cart.item.added`.

### 7.4 Idempotency — three layers
| Where | Mechanism |
|---|---|
| order-service handlers | state guard (only act if the order is in the expected state) |
| inventory | `processed_events` PK on eventId |
| payment | `payments` PK is the eventId |
| analytics | Mongo `_id = orderId:status` → upsert |

The key idea: **retries are safe only if applying the same message twice has the same effect as applying it once.**

### 7.5 Ordering and partitioning
- Order is only guaranteed within a partition. Key = orderId → one order's messages stay ordered.
- A consumer group gets at most one consumer per partition → partitions cap parallelism (3 partitions → max 3 useful inventory instances).
- Adding partitions later changes which partition a key maps to: messages for in-flight orders can end up on different partitions. Size partitions up front.
- No ordering across topics. That's fine here because order-service never sends release until it has received RESERVED.

### 7.6 Polyglot persistence
| Store | Used for | Why this and not the others |
|---|---|---|
| Postgres | orders, stock, payments, product master | transactions, constraints, conditional updates |
| Elasticsearch | product search | inverted index, relevance, fuzzy matching |
| Redis | carts | fast, simple hash ops, disposable data |
| MongoDB | analytics events | write-once documents with nested item arrays, aggregation pipeline, no joins |

### 7.7 The catalog dual write (expect this "gotcha")
**Q: "Your catalog writes to Postgres and then Elasticsearch. Isn't that the dual-write problem you just told me about?"**
"Yes, it is, and I accepted it deliberately. If the Elasticsearch write fails, the product exists but isn't searchable until it's re-indexed. The search index is derived data I can always rebuild from Postgres, so the worst case is a stale search result, not lost money or stock. If it mattered, I'd use the same outbox, or CDC with Debezium, to feed an indexer."

This answer shows you apply the heavy pattern where it's needed, not everywhere.

### 7.8 Analytics: Kafka Streams + MongoDB
- **Plain consumer → MongoDB**: stores every raw outcome durably for historical queries (revenue by product via `$unwind` + `$group`).
- **Kafka Streams topology**: merges `order.confirmed` and `order.failed`, re-keys by status, counts per 1-minute tumbling window (with a grace period for late events), materializes a state store. The REST API reads it directly (**interactive queries**): no database round trip.
- Why two consumers: blocking MongoDB writes inside a Streams processor would stall the stream thread.
- Caveat: with several analytics instances, each only holds its own partitions' state → route queries using `KafkaStreams#queryMetadataForKey`.

### 7.9 Monitoring
- **Consumer lag** per group (kafka-exporter → Prometheus → Grafana). Rising lag = a consumer is slow or down.
- Throughput per topic, HTTP rate and latency per service (Spring Actuator + Micrometer), JVM heap.
- Business signals worth adding: orders stuck in a non-terminal state for more than N minutes; outbox rows unpublished for more than N seconds.

### 7.10 Docker and Kubernetes
- One multi-stage Dockerfile per service (Maven build stage → slim JRE runtime).
- Compose for local; K8s: Deployment + Service per app, a Job to create topics, Secrets for DB credentials, PVCs for Postgres/Mongo, readiness probes on `/actuator/health`.
- Explicit topic creation (auto-create disabled) so partition counts are a decision, not an accident.

---

## 8. Question bank

### A. Kafka fundamentals

**Q1. What's a topic, a partition, an offset, a consumer group?**
Topic: named log. Partition: an ordered, append-only shard of the topic, the unit of parallelism and ordering. Offset: position of a record in a partition. Consumer group: consumers sharing a group id split the partitions between them; each partition is read by exactly one member.

**Q2. How does Kafka guarantee ordering?**
Only within a partition. I key by orderId so all messages for one order land in the same partition.

**Q3. What's a rebalance and why does it matter?**
When consumers join or leave a group, partitions are reassigned. During it, consumption pauses, and uncommitted work can be redelivered to the new owner. That's one reason consumers must be idempotent.

**Q4. acks=0/1/all?**
0: don't wait. 1: leader wrote it. all: all in-sync replicas wrote it. With `min.insync.replicas=2` and replication factor 3, `acks=all` tolerates one broker loss without losing acknowledged writes.

**Q5. What's the ISR?**
In-sync replicas: followers caught up with the leader. Only an ISR member can be elected leader without data loss.

**Q6. At-most / at-least / exactly-once?**
See 7.3. Most of my system is at-least-once + idempotent consumers; inventory and payment use Kafka transactions for their consume-produce step.

**Q7. What does ZooKeeper do? Do you still need it?**
In older Kafka it stores cluster metadata and elects the controller. KRaft replaces it with a Raft quorum inside Kafka; KRaft is production-ready since 3.3 and ZooKeeper support was removed in Kafka 4.0. I used ZooKeeper-based images locally; a new deployment should use KRaft.

**Q8. Retention vs compaction?**
Retention deletes by time/size. Compaction keeps the latest record per key, useful for "current state" topics (and the Streams changelog).

**Q9. How many partitions and why 3?**
Parallelism cap for consumers. 3 is a local default; in production I'd size by target throughput divided by per-consumer throughput, with headroom, since changing it later remaps keys.

**Q10. What is consumer lag?**
Latest offset minus committed offset per partition. The most useful single health signal for event-driven services.

**Q11. Auto-commit vs manual commit?**
Auto-commit can commit before processing finishes (message loss) or after a crash-reprocess (duplicates). Transactional services commit offsets inside the Kafka transaction; the rest commit after processing.

**Q12. How do you handle a poison message?**
Retry with backoff, then route to a dead-letter topic so the partition keeps moving; alert and inspect. Never let one bad record block everything behind it.

### B. Design

**Q13. Walk me through a checkout.** → Section 5.3, with the state machine.

**Q14. Why a saga instead of a distributed transaction (2PC)?**
Each service has its own DB; 2PC needs all of them to hold locks until a coordinator decides, it blocks if the coordinator dies, and Kafka doesn't participate in XA. Sagas trade immediate consistency for availability and use compensations.

**Q15. What's a compensating transaction? Isn't it just a rollback?**
No. A rollback makes it as if nothing happened. A compensation is a new forward action that semantically undoes the effect. The reservation really existed and other orders really saw less stock for a moment; releasing it restores a correct end state. Eventually consistent, not atomic.

**Q16. Why orchestration and not choreography?** → 7.1.

**Q17. Why the outbox if you have Kafka transactions?** → Kafka transactions can't include Postgres. Different problem, different tool (7.2 vs 7.3).

**Q18. Why does checkout return 202 instead of 200?**
The request is accepted but not finished; the saga completes asynchronously. The client polls the order status (or could get a push via WebSocket/SSE).

**Q19. Why does order-service fetch prices from catalog instead of trusting the cart?**
Never trust client-supplied prices. The server looks up the price at checkout and snapshots it into `order_items`, so later price changes don't rewrite history.

**Q20. Why does order-service call catalog over HTTP instead of Kafka?**
It's a read with nothing to compensate. Going async would add complexity for no benefit. The trade-off: catalog being down blocks checkout, which a local price cache or a replicated price table (fed by product events) would fix.

**Q21. Why commands with a `.requested` / `.result` naming convention?**
Makes intent explicit: a command has exactly one handler and expects an answer; an event is a broadcast fact.

**Q22. What's in `order.confirmed` and why so much?**
The full order, so consumers don't need to call back to order-service (event-carried state transfer). Keeps services decoupled at read time.

### C. Failure and consistency → mostly section 6

**Q23. What if the compensating action fails?** (#9)
**Q24. What if the same message arrives twice?** (#2, #5, #6 and 7.4)
**Q25. What if order-service crashes mid-saga?** Its state is in Postgres and its pending messages are in the outbox; on restart the poller resumes and incoming results are handled against the stored state.
**Q26. What if payment times out vs declines?** (#13)
**Q27. Can a customer see stock go down and come back up?** Yes, briefly, during a reserve then release. That's eventual consistency; it's acceptable for stock, and I'd explain why it's not acceptable for, say, a bank balance shown as final.
**Q28. Could two orders both get the last item?** No (#12).

### D. Data

**Q29. Draw your tables.** → Section 3.
**Q30. Why UUIDs instead of auto-increment ids?** Generated in the service without a DB round trip, globally unique across services, safe to use as a message key before the row is committed.
**Q31. Why `BigDecimal` for price but `double` in Elasticsearch?** Exact arithmetic for money in the source of truth; ES needs a numeric type for sorting and ranges, and search results aren't used for billing.
**Q32. Why MongoDB for analytics?** Write-once, nested items, ad-hoc aggregation, no relational integrity needed.
**Q33. What indexes would you add?** `orders(user_id, created_at)` for "my orders", `orders(status, updated_at)` for a stuck-saga sweeper, partial index on `outbox_events(created_at) WHERE published = false` for the poller.
**Q34. How do you clean up `processed_events` and `outbox_events`?** Delete published outbox rows and old ledger rows after a retention window longer than the maximum redelivery window.

### E. Scaling and performance

**Q35. How would you scale this to 10× traffic?** More partitions and consumer instances for inventory/payment; multiple order-service replicas with `SKIP LOCKED` in the poller (or CDC); read replicas or caching for catalog; ES cluster for search; measure consumer lag to find the bottleneck first.
**Q36. What's your bottleneck?** Likely the hot-product row in `inventory_items` under a flash sale (everyone decrements the same row). Options: bucketed stock counters, or pre-allocating stock to shards.
**Q37. What does EOS cost you?** Throughput and latency; that's why cart telemetry doesn't use it.
**Q38. Why a 300 ms outbox poll?** Latency vs DB load trade-off; CDC removes polling entirely.

### F. Operations

**Q39. What do you monitor and alert on?** Consumer lag, error rates, DLQ size, stuck sagas, unpublished outbox age.
**Q40. How do you trace one order across services?** Today: orderId in every log line and message key. Better: OpenTelemetry trace context propagated in Kafka headers.
**Q41. How do you deploy a new version without losing messages?** Rolling update; consumers commit only processed offsets; a rebalance hands partitions to the new pods; idempotency absorbs redeliveries.
**Q42. How do you change a message schema safely?** Add fields only (backward compatible), tolerate unknown fields on read (`@JsonIgnoreProperties(ignoreUnknown = true)`); for strictness, Avro/Protobuf with a Schema Registry enforcing compatibility.

### G. Spring Boot specifics (you'll be asked about code you wrote)

**Q43. How does `ProductRepository` work without an implementation?** Spring Data generates a proxy at startup from `JpaRepository<Product, String>`; derived queries are built from method names (like your `findByNameContainingIgnoreCaseOrDescriptionContainingIgnoreCase`).
**Q44. Why controller → service → repository?** HTTP concerns, business logic, and data access separated; service logic is reusable from a Kafka listener and testable without the web layer.
**Q45. What does `@RequiredArgsConstructor` do?** Lombok generates a constructor for `final` fields; Spring injects dependencies through it (constructor injection).
**Q46. What did the "Multiple Spring Data modules found" log mean?** With JPA and Elasticsearch both on the classpath, Spring uses the entity annotation (`@Entity` vs `@Document`) and the repository base interface to decide which store each repository belongs to.
**Q47. `ddl-auto: update` in production?** No; use versioned migrations (Flyway/Liquibase).
**Q48. `@Transactional` and Kafka transactions together?** Two transaction managers (JPA and Kafka), two separate commits; the order of commits and the idempotency ledger handle the gap (scenario 5).

### H. About you and the project

**Q49. What was the hardest part?** Pick a real one from building it. Candidates so far: understanding why the outbox is needed even with Kafka transactions; the out-of-stock path that must commit on Kafka but roll back in the DB; environment problems like Windows reserving the Postgres port range.
**Q50. What would you do differently?** Section 9.
**Q51. How does this relate to your work experience?** PayPal: high-volume payment events and downstream propagation. OLA: repayment service with asynchronous processing and a reconciliation job for stuck transactions, which is the real-world version of scenario 13.

---

## 9. Known limitations — say these before they ask

1. **No saga timeout**: an order can wait forever if a participant never answers. Fix: a scheduled sweeper that fails or compensates sagas older than N minutes.
2. **Cart cleared at checkout** even if the order later fails. Fix: frontend offers "re-add items", or cart listens to `order.failed`.
3. **Catalog dual write** to ES isn't atomic (7.7).
4. **Synchronous price lookup** couples checkout to catalog availability.
5. **No auth**: user id is a path parameter. Real system: OAuth2/JWT at a gateway.
6. **No schema registry**: JSON contracts duplicated as DTOs in each service.
7. **Single broker, RF=1 locally**: no fault tolerance in the dev setup.
8. **Payment is simulated**: no gateway idempotency keys or reconciliation.
9. **ZooKeeper instead of KRaft.**

Naming limitations yourself signals senior judgment; being caught not knowing them signals the opposite.

---

## 10. Build status — keep this honest and up to date

| Step | Component | Status |
|---|---|---|
| 1 | catalog-service: Postgres CRUD | ✅ running locally |
| 2 | catalog-service: Elasticsearch search | ✅ running; end-to-end POST → search still to verify |
| 3 | inventory-service: first Kafka producer/consumer | ⬜ next |
| 4 | inventory-service: exactly-once | ⬜ |
| 5 | payment-service | ⬜ |
| 6 | order-service: saga + outbox | ⬜ |
| 7 | cart-service: Redis | ⬜ |
| 8 | analytics-service: Streams + MongoDB | ⬜ |
| 9 | Compose → Kubernetes → Prometheus/Grafana | ⬜ |

Until a row is ✅, talk about it as designed: "the design is X; I've built A and B, and I'm implementing C now." Interviewers generally respond well to that. They respond badly to a claim that falls apart at "can you show me?".

---

## 11. Last-minute cheat sheet

- **Saga** = sequence of local transactions + compensations. Orchestrated by order-service.
- **Happy path**: reserve → charge → confirm.
- **Compensation**: charge declined → release stock → FAILED.
- **Outbox** = state change + message in one DB transaction; poller publishes; at-least-once.
- **EOS** = idempotent producer + transactional producer + offsets in transaction + `read_committed`. Only inside Kafka.
- **Idempotency** = eventId ledger (inventory), eventId PK (payment), state guard (order), `_id` upsert (analytics).
- **Ordering** = key by orderId, per-partition order.
- **Dual write** = the core problem the outbox solves; accepted deliberately for the ES search index.
- **Stores** = Postgres (truth), ES (search), Redis (cart), Mongo (analytics).
- **Monitor** = consumer lag first.
- **Limitations** = no saga timeout, cart cleared early, sync price lookup, no auth, no schema registry.
