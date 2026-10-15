
# Real-time Updates (Revision Notes)

Real-time updates are two separate problems:

1. **Server to client:** how the update reaches the user's device.
2. **Source to server:** how the server that holds the user's connection learns about the update.

---

## 1. Server to Client

| Method | How it works | Use when | Watch out for |
|---|---|---|---|
| **Simple polling** | Client asks every few seconds | Latency of 2–5 s is fine | Wasted requests; 1M clients every 10 s = 100k reads/s |
| **Long polling** | Server holds the request until data arrives, then client asks again | Infrequent updates, e.g. waiting for a payment result | Extra latency when updates come close together; load balancer timeouts |
| **SSE** | One HTTP response stays open and streams chunks | One-way server push: dashboards, AI token streaming | Some proxies buffer streams; client must reconnect with last event ID |
| **WebSocket** | HTTP upgrades to a two-way persistent connection | Frequent messages both ways: chat, games | Stateful; needs L4 load balancer, reconnect logic, careful deploys |
| **WebRTC** | Direct peer-to-peer after a signaling server introduces peers | Audio/video calls, screen sharing | Complex: signaling server, STUN (find public address), TURN (relay fallback) |

**Decision order:** not latency-sensitive → polling. One-way → SSE. Frequent two-way → WebSocket. Audio/video or peer-to-peer → WebRTC.

Common combo: SSE for receiving updates, plain HTTP POST for sending.

### Networking basics
- **TCP:** connection-based, reliable and ordered, costs a handshake.
- **UDP:** no connection, no delivery or order guarantee, fast.
- **L4 load balancer:** routes by IP and port and keeps the same TCP connection to one server, so it suits WebSockets.
- **L7 load balancer:** reads HTTP and can route by URL or cookie, but opens new connections to backends, so it suits normal HTTP and long polling.

### WebSocket in practice
- Put a dedicated WebSocket service right behind an L4 load balancer and keep the rest of the system stateless.
- Use "least connections" load balancing.
- On deploys, drop connections and let clients reconnect.

---

## 2. Source to Server

| Approach | How it works | Use when |
|---|---|---|
| **Pull (polling the DB)** | Updates are written to a DB; clients poll it | Real-time isn't really needed |
| **Consistent hashing** | Each user is always assigned to one server (via ZooKeeper/etcd); the sender hashes the user ID to find it | Each connection holds heavy state, e.g. an open Google Doc |
| **Pub/Sub** | Endpoint servers subscribe to topics (Kafka/Redis); updates are published to the topic | Default choice; many clients, little state per connection |

- **Simple modulo hashing:** changing the number of servers moves almost every user. **Consistent hashing** moves only the users on the affected part of the ring.
- **Scaling with consistent hashing:** record old and new assignments, move clients gradually, send messages to both servers until the move is done.
- **Pub/Sub downsides:** the broker is a bottleneck and single point of failure (scale with Redis Cluster), and it doesn't know when clients disconnect.

---

## Common Deep Dives

- **Reconnection:** use heartbeats to catch dead connections; give messages sequence numbers or keep a per-user queue (e.g. Redis Streams) so clients can fetch what they missed.
- **Celebrity / huge fan-out:** cache the update once and spread it through layers (regional or broadcast nodes) instead of writing to millions of feeds. See hierarchical aggregation in the Scaling Writes notes.
- **Message ordering:** in product interviews, send related messages through a single server or partition and timestamp them there. Vector clocks are for deep infrastructure questions.

---

## Interview Tips

- Mention real-time needs early, before the interviewer asks.
- Don't over-engineer: suggest polling if it's good enough, and check with the interviewer first.
- Defaults: **SSE or WebSocket** for the client side, **Pub/Sub** for the server side. Use **consistent hashing** only when per-connection state is heavy.
