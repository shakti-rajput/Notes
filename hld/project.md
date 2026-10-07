# Interview Notes: Availability and API Performance

## 1. How do you define availability for your service?

**Definition**

Availability is the percentage of requests that succeed.

```
Availability = successful requests ÷ total requests
```

If 1,000 requests come in and 999 work, availability is 99.9%.

| Availability | Downtime per month |
|---|---|
| 99.9% | about 43 minutes |
| 99.99% | about 4 minutes |

**Short answer**

- **Measured:** "We tracked error rate and latency on dashboards, with alerts when errors went up."
- **Kept high:** "We ran multiple instances behind a load balancer, so if one died the others kept serving."

---

## 2. How did you make sure, and measure, that the system is highly available?

### How I made sure (design)

- **No single point of failure:** several instances of the service behind a load balancer. If one dies, the others keep serving.
- **Health checks:** the load balancer stops sending traffic to an unhealthy instance automatically.
- **Queue for slow or risky work:** the payment gateway call went through SQS.

### How I measured it

- **Error rate:** the percentage of requests returning 5xx errors.
- **Latency:** p95 or p99 response time, because a very slow response is as bad as a failed one.
- **Queue monitoring:** CloudWatch metrics for SQS (queue depth, message age).

**Tools:** "At OLA we used CloudWatch. For SQS I watched queue depth and message age."

**Example alert thresholds**

- 5xx rate above 1% for five minutes
- Oldest SQS message older than ten minutes

---

## 3. How do you decide which server to hit when there are multiple instances?

The load balancer decides, not the client.

| Method | How it works | Good for |
|---|---|---|
| Round robin | Sends requests to each server in turn: 1, 2, 3, 1, 2, 3 | Servers of equal size, similar requests |

**Two things that make it work**

- **Health checks:** the load balancer pings each instance regularly and stops sending traffic to any that fail.
- **Stateless service:** no user data is kept in the server's memory. Sessions and shared data live in Redis or the database, so any server can handle any request.

---

## 4. How do you make your REST API faster?

### Levers

- **Asynchronous processing:** save the request, reply quickly, and let a background worker do the slow work through a queue.
- **Redis caching:** for data that is read often.
- **Database:** indexes on the columns we searched by.
- **External calls:** timeouts, retries with backoff, and idempotency keys so a retry never double-charges.
- **Horizontal scaling:** stateless instances behind a load balancer, which is also the high-availability story.
- **Measurement:** load testing before release and dashboards for latency and error rate.

### Tell it as a short story in three steps

1. **What was slow.** "The slow part was calling the payment gateway. Our API had to wait for it."
2. **What I did.** "So I stopped making the user wait. The API saves the request, replies straight away, and a background worker talks to the gateway through a queue (SQS). I also added database indexes on the columns we searched by."
3. **What happened.** "The API became fast and stopped timing out at busy times."
