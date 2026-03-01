# Caching & Edge Delivery — Distributed Cache + CDN

> Component #5 from the High-Value System Design Building Blocks.
> Problem Solved: Cut latency and DB load; global fast delivery.
> Core Mechanic: Multi-layer caches (edge/CDN/app/DB); TTLs/invalidation; cache keys; static assets at edge.

---

## 1. What to Cache — The Decision Framework

Before architecting a solution, you need a clear mental model for what data is a good candidate for caching. The framework is a multi-dimensional evaluation across four axes.

### The Four Axes

**Axis 1: Read/Write Ratio — The Golden Rule**

This is the most important factor. Caching is most effective for data that is read frequently but written infrequently.

- Excellent candidates (High Read / Low Write): Product details, static assets (CSS/JS/fonts), category pages. A popular product might be read millions of times a day but updated once a month.
- Poor candidates (Low Read / High Write): Live shopping cart (updated every add/remove, only read by one user), real-time stock tickers.

**Axis 2: Cost of Generation — The "Bang for Your Buck" Metric**

If data is computationally expensive to generate, it's a strong candidate even if not read as frequently.

- Excellent candidates (High Cost): User-specific recommendations (output of ML models or complex multi-table joins), aggregated data like "Top 10 Bestsellers" (requires scanning and aggregating large datasets), complex product detail pages joining 5+ tables.
- Poor candidates (Low Cost): Simple primary key lookups. At moderate scale, the caching overhead may exceed the benefit.

**Axis 3: Staleness Tolerance — "How Fresh is Fresh Enough?"**

This determines your TTL strategy. The business must define acceptable staleness per data type.

- High tolerance: Product images, descriptions (days/weeks), user reviews (minutes/hours).
- Low tolerance: Product price (short TTL of 30s–5min is a common compromise), inventory/stock count (very short TTL of 5–10s, or cache a status like "high/low/out" instead of exact count).
- Zero tolerance: Payment transaction status, final checkout summary. These must never be cached.

**Axis 4: Scope — "Who Is This For?"**

This determines *where* you cache, not just *whether* you cache.

- Global/Public data (same for every user): Product pages, images, static assets. Perfect for CDN caching at the edge.
- User-Specific/Private data: Recommendations, order history, "recently viewed." Cannot go on a public CDN. Use a centralized cache like Redis keyed by `user_id`.

### The Decision Matrix

| Data Type | Read/Write | Cost to Generate | Staleness Tolerance | Scope | Verdict |
|---|---|---|---|---|---|
| Product Images, JS/CSS | Very High Read | Low | Very High | Global | Excellent. CDN with long TTL. |
| Product Details (Desc, Specs) | Very High Read | Medium (DB join) | High | Global | Excellent. CDN + central cache. |
| Product Price | Very High Read | Low | Low | Global | Good, short TTL (1–5 min). |
| Inventory Count | High Read | Low | Very Low | Global | Tricky. Very short TTL (10s) or skip. |
| User Recommendations | High Read | Very High (ML/DB) | Medium | User-Specific | Excellent. Redis keyed by user_id. |
| Shopping Cart | Low Read / High Write | Low | Zero | User-Specific | Poor. Generally should not be cached. |
| "Top 10" Lists | Very High Read | High (Aggregation) | Medium | Global | Excellent. Central cache, regenerate on schedule. |

---

## 2. The Multi-Layer Caching Architecture

A user request traverses layers in sequence. A cache miss at one layer causes the request to fall through to the next.

```
User's Device
    → [Layer 1: Browser Cache]
        → Internet
            → [Layer 2: CDN Edge Cache]
                → Origin Data Center
                    → [Layer 3: Application-Level Cache (Redis)]
                        → [Layer 4: Database Buffer Pool]
                            → Database Disk
```


### Layer 1: Browser Cache

**What it caches:** Static assets (CSS, JS, fonts, site-wide images), product images after first view.

**Technology:** Standard HTTP caching headers.
- `Cache-Control: public, max-age=31536000, immutable` — for versioned/hashed assets.
- `ETag` — for validation. Browser sends its ETag; server responds `304 Not Modified` if unchanged, saving bandwidth.

**TTL Strategy:**
- Versioned assets use content-hash filenames (e.g., `app.a1b2c3d4.js`). When the file changes, the name changes. This allows a 1-year `max-age`. Users always get the new version on deploy, and cache it indefinitely otherwise. This is called cache busting.
- Product images: `Cache-Control: public, max-age=86400` (24 hours).

**Failure Mode:** Extremely resilient. If the user clears their cache, the browser simply makes a new request to the CDN. No system-wide impact, just a slightly slower first load for that user.

---

### Layer 2: CDN (Content Delivery Network)

The most critical layer for solving global latency (EU/SEA users) and offloading origin traffic.

**What it caches:** Static assets, product images, public API responses (`GET /api/products/{id}`), full server-rendered HTML pages.

**Technology:** AWS CloudFront, Cloudflare, or Fastly. Provider selection matters — Cloudflare/Fastly purges propagate globally in 2–5 seconds. CloudFront can take 10–30 seconds. For tight invalidation SLAs, provider choice is a real architectural decision.

**TTL Strategy:**
- Static assets: 1 year, relying on cache-busting filenames.
- Product images: 7–30 days, with targeted CDN API purges on update.
- Product detail pages / API responses: Short TTL (e.g., 2 minutes). This drastically reduces origin traffic while keeping data reasonably fresh. 99% of requests are served from the edge.
- Advanced — Cache Tags: Tag the cached page for product 123 with `product-123`. When the backend updates that product's price, one API call purges all content tagged `product-123` across all PoPs globally. More precise and efficient than URL-based purges. Supported by Fastly and Cloudflare.

**Failure Mode:** A full CDN outage is a Major Incident. All traffic routes directly to origin, immediately triggering DB CPU spikes and likely a full site outage. Mitigation: have a DNS-based failover plan, and ensure origin infrastructure has robust rate limiting and load shedding to survive the initial flood.

---

### Layer 3: Application-Level Cache (Redis)

Sits within the data center. Protects the database. Caches expensive computations and user-specific data.

**What it caches:** User-specific recommendation results, aggregated data ("Top 10 Bestsellers"), database query results (second line of defense for product details), session data.

**Technology:** Redis. Rich data structures (Hashes, Sorted Sets) map naturally to use cases — user profiles in a Hash, "recently viewed items" in a Sorted Set. Run as a managed service (AWS ElastiCache for Redis) with Multi-AZ configuration for automatic failover.

**TTL Strategy:**
- User recommendations: 1–4 hours, keyed by `user_id`.
- "Top 10" lists: Generated by a background job every 15 minutes. TTL set to ~20 minutes to ensure no gap. This is a cache warming strategy.
- DB query results: 5–10 minutes depending on volatility.

**Failure Mode:** A Redis outage is a Critical Incident. All requests that relied on the cache now hit the database directly, causing the exact CPU spike problem the cache was designed to prevent. Mitigation: Multi-AZ provides automatic failover (30–60 second window). Application code must use a circuit breaker — if the cache is down, fail fast or degrade gracefully (e.g., render the page without the recommendations section) rather than falling through to the DB for every request.

---

### Layer 4: Database Buffer Pool

The final line of defense, managed internally by the database engine.

**What it caches:** Frequently accessed data blocks from disk, query execution plans.

**Technology:** Internal to the database (e.g., InnoDB Buffer Pool in MySQL, shared_buffers in PostgreSQL).

**TTL Strategy:** Managed automatically by the database's internal LRU-variant algorithm. The engineer's job is not to set TTLs but to properly provision and tune — allocate 70–80% of server RAM to the buffer pool.

**Failure Mode:** This doesn't "fail" in the traditional sense. The failure mode is an undersized or poorly configured cache. If the buffer pool is too small, the hit rate is low, forcing constant disk reads, resulting in high I/O wait and slow queries. Monitor the buffer pool hit ratio as a key operational metric.

---

## 3. Cache Invalidation in Practice — The Flash Sale Scenario

Cache invalidation is one of the hardest problems in distributed systems. The flash sale scenario makes it concrete: 500 products drop in price simultaneously at 12:00 PM, and users must see the new prices within 30 seconds.

### Pre-Scheduling (The Right Approach)

Price updates are not triggered by a human clicking a button at noon. They are pre-scheduled and loaded into a durable message queue (AWS SQS or similar). Each message contains `product_id`, `new_price`, and `sale_start_time`. A pool of workers begins processing this queue at exactly 12:00:00 PM. This decouples the trigger from the execution and provides reliability via retry semantics.

### The End-to-End Invalidation Flow

**T+0s to T+5s — DB Write + Cache Invalidation**

For each product update, a worker performs these steps in strict order:

1. Execute `UPDATE products SET price = :new_price WHERE id = :product_id` and commit. This is the source of truth.
2. Immediately issue `DEL cache:product:123` to Redis. This purges the Layer 3 application cache.
3. Do NOT immediately call the CDN API. Doing 500 separate API calls would be slow and risk rate-limiting. Instead, collect the product's cache tag (e.g., `product-123`) into a batch list.
4. Once a batch of ~25 products is collected (or after a 1-second timeout), make a single CDN API call: `POST /api/v4/purge` with payload `{"tags": ["product-123", "product-124", ...]}`.

**T+5s to T+10s — Global CDN Propagation**

The CDN's control plane propagates the invalidation command globally to all PoPs. Cloudflare/Fastly: 2–5 seconds. CloudFront: 10–30 seconds. The cache entry for the product page and API response is now marked invalid in the Singapore PoP.

**T+10s — User Request in Singapore**

A user in Singapore requests the product page. The CDN PoP finds the entry is purged — cache miss at the edge. The PoP forwards the request to origin over the CDN's optimized backbone network.

**T+10s to T+10.3s — Origin Fetch**

The application checks Redis — miss (we invalidated it). The application queries the database, gets the new price, renders the full HTML page, and sends it back. The response includes the header `Cache-Tag: product-123`.

**T+10.3s to T+10.5s — Re-caching and Delivery**

The Singapore PoP stores the fresh copy, associates it with the `product-123` tag and the 2-minute TTL, and streams the response to the user.

Result: The user in Singapore sees the correct flash sale price approximately 10–15 seconds after the sale began, well within a 30-second SLA.

---

### Failure Points and Mitigations

**Failure 1: DB Overloaded During Bulk Updates**
- Problem: 20 workers processing 500 products simultaneously could overload the database.
- Mitigation: Queue throttling (control worker concurrency), connection pooling, idempotent retry logic. SQS visibility timeout ensures failed messages are retried. A Dead-Letter Queue (DLQ) catches messages that fail repeatedly for manual inspection.

**Failure 2: CDN Purge API Fails**
- Problem: DB is updated, but the CDN API returns a 5xx. The CDN continues serving stale pre-sale prices.
- Mitigation: Retry with exponential backoff. If purge fails after 3–4 retries, log the failed tags to a high-priority alert channel (PagerDuty) AND write them to a separate retry queue. The short 2-minute TTL is the ultimate safety net — even if a purge fails completely, stale content expires automatically within 2 minutes.

**Failure 3: The "Split Brain" — DB Write Succeeds, Invalidation Never Fires**
- Problem: The worker commits the DB transaction, then the process crashes before issuing the CDN or Redis invalidation. This is the most insidious failure because the DB is correct but the cache is permanently stale until TTL expiry.
- Mitigation — Change Data Capture (CDC): The gold standard. Use Debezium to read the database's transaction log (WAL). When it detects a change in the `products` table's `price` column, it emits an event to a Kafka topic. A dedicated, highly-available "Invalidator Service" consumes from this topic and issues purge commands to the CDN and Redis. The application's only responsibility is writing to the database. Any committed change will eventually trigger an invalidation event, even if the main application crashes. This moves from application-level "at-least-once" retries to a reliable, event-driven architecture.

---

## 4. Cache Stampede — The Thundering Herd Problem

### What Is a Cache Stampede?

When a popular cache key expires (or is purged), many concurrent requests simultaneously find a cache miss and all attempt to regenerate the data from the database at the same time. This creates a sudden spike of expensive DB queries.

### The Naive Fix: SETNX Mutex — Why It Fails at Scale

The `SETNX` (SET if Not eXists) approach acquires a lock so only one process regenerates the data while others wait. This works at small scale but catastrophically fails under a true thundering herd.

**The failure scenario with 50,000 concurrent users hitting a cold cache:**

1. The CDN PoP, with its cache for Product X now empty, forwards all 50,000 requests to origin (no request collapsing configured).
2. Application servers (say 10 servers × 100 workers = 1,000 total capacity) accept the first 1,000 requests. The other 49,000 queue in TCP listen buffers.
3. Worker A acquires the `SETNX` lock and proceeds to the database (~200ms query).
4. The other 999 workers fail to acquire the lock and enter a spin-wait loop: `sleep(50ms)`, `check lock`, `sleep(50ms)`. These workers are fully occupied — they cannot serve any other traffic.
5. All 1,000 workers are now tied up. New requests (for other products, checkout, health checks) are rejected or queue indefinitely.
6. The load balancer's health checks start failing. It marks app servers as unhealthy and removes them from rotation.
7. Result: A self-inflicted Denial of Service. The database is protected, but the entire application tier is sacrificed.

**The correct fix is at the layer before the application servers:**

- CDN Request Collapsing (best solution): Cloudflare's "Cache Lock," Fastly's "Request Collapsing," Varnish's "Grace Mode." The CDN holds 49,999 requests at the edge, sends one to origin, and fans the response out to all waiting clients. The application never sees the stampede.
- API Gateway / Proxy Layer: NGINX or Envoy can be configured to perform request collapsing before requests reach application code.

---

### The Production-Grade Fix: Probabilistic Early Expiration (XFetch Algorithm)

This pattern eliminates the "hard expiration" event that creates the vulnerability window in the first place. It is based on the XFetch algorithm (Vattani et al., 2015).

**How it works mechanically:**

Instead of storing just the value, store a richer object in Redis:

```
KEY: cache:product:X
VALUE: {
  "data": "<html>...</html>",
  "created_at": 1678881600,   // Unix timestamp when cached
  "ttl": 120                  // Full TTL in seconds
}
```

On every request for Product X:

1. Fetch the full object from Redis.
2. Immediately serve `object.data` to the user. The user gets a fast response. The data may be slightly stale, but it is served instantly.
3. After serving, check whether this request should trigger a background refresh using the formula:

```
current_time - (delta * log(rand())) >= created_at + ttl
```

Where:
- `delta` is a configurable window (e.g., 10 seconds) representing how early you're willing to start refreshing.
- `rand()` is a random float between 0.0 and 1.0.
- `log(rand())` produces a value from -infinity to 0, so `-(delta * log(rand()))` is a random positive number with an exponential distribution — small values are far more likely than large ones.

4. As `current_time` approaches `created_at + ttl`, the probability of satisfying the condition increases exponentially. One request is eventually "chosen."
5. The chosen request attempts to acquire a short-lived refresh lock (`SETNX product:X:refresh_lock 1 EX 10`). If it gets the lock, it re-fetches from the database and writes a new object with an updated `created_at`. If it fails to get the lock, another request was chosen moments before — it does nothing and exits.

**Why this is better than the mutex:**

| Dimension | SETNX Mutex | Probabilistic Early Expiration |
|---|---|---|
| Hard expiration event | Yes — creates a vulnerability window | No — cache is continuously refreshed before expiry |
| User latency during miss | All 50,000 users wait | Only one user triggers a background refresh; all others get stale data instantly |
| DB load pattern | Sharp spike every TTL interval | Low, constant hum of single background refreshes |
| Application server impact | Connection pool starvation | No impact — workers are never blocked waiting |

The mutex is a single-threaded gatekeeper that causes a pile-up. Probabilistic early expiration is like having a crowd where one random person is occasionally tapped on the shoulder to quietly fetch a new round while the party continues uninterrupted.

---

## 5. Redis Failover — The Math of Cache Dependency

### The Scenario

Redis primary dies. ElastiCache promotes a replica. Failover takes 30–60 seconds. Application is configured to "fail open" — cache misses fall through to the database.

Platform: 500K concurrent users at 2:00 AM. Normal cache hit rate: 95%. Database capacity: 5,000 QPS before degradation.

### The Math

**Step 1: Calculate total platform RPS**

500K concurrent users, each performing a cache-dependent action every ~10 seconds (conservative think time for an e-commerce site):

```
Total RPS = 500,000 / 10 = 50,000 RPS
```

**Step 2: Normal database load (baseline)**

```
Normal DB Load = 50,000 RPS × 5% miss rate = 2,500 QPS
```

This is healthy — 50% of the 5,000 QPS capacity, with a good buffer.

**Step 3: Database load during Redis failover**

With Redis unavailable, cache hit rate drops to 0%. All 95% of requests that would have been served by Redis now fall through:

```
New load from cache misses = 50,000 × 95% = 47,500 QPS
Total DB load = 2,500 + 47,500 = 50,000 QPS
```

**Step 4: The verdict**

```
50,000 QPS / 5,000 QPS capacity = 10x overload
```

The database instantly exhausts its connection pool. Queries time out. CPU spikes to 100%. Application servers waiting on DB connections exhaust their own worker pools. The site goes down.

"Fail open to DB" is a naive and dangerous strategy for a critical dependency like a cache.

---

### Mitigation 1: Circuit Breaker

Wrap the Redis client in a circuit breaker (Resilience4j in Java, Polly in .NET, or a custom implementation).

- Closed state (normal): Requests pass through. Monitor for failure threshold (e.g., >50% timeouts over a 5-second window).
- Open state (failure): When threshold is breached, the breaker trips. For the next 60 seconds, any call to Redis fails fast without attempting a connection. The application does NOT fall back to the database. Instead:
  - For non-essential data (recommendations): Return an empty set. Page renders without that component.
  - For essential data (product page): Return a `503 Service Unavailable` with a friendly error page. This sheds load instantly and protects the system.
- Half-open state (recovery): After 60 seconds, allow one test request. If it succeeds (failover complete), close the breaker. If it fails, stay open.

The site may be partially degraded or show errors for 60 seconds, but it prevents a full system meltdown and allows quick recovery.

### Mitigation 2: Read from Replica (Architecturally Superior)

Since you're already paying for Redis replicas for high availability, use them during failover.

- Configure the Redis client to be aware of both the primary endpoint and the reader endpoint (AWS ElastiCache provides a Reader Endpoint that load balances across available replicas).
- Normal operation: Write to primary, read from primary for strongest consistency.
- During failover: If a read from the primary fails or times out, immediately retry against the Reader Endpoint. Use a very short connection timeout (50–100ms) rather than waiting for a full request timeout before retrying.

Why this is better: During the 60-second failover, replicas are still up and running with data that is only milliseconds stale. For 99% of use cases (product info, categories, non-critical user data), serving slightly stale data is infinitely better than returning an error. The replicas continue to absorb the 47,500 RPS of read traffic. The database never sees the spike.

**Combined strategy:** Use "Read from Replica" as the primary defense. Layer a circuit breaker on the entire Redis interaction (primary + replicas) as a final backstop for a total cluster-wide outage.

---

## 6. Cache Key Design and Data Ownership

### The Race Condition in Shared Cache Keys

When two independent services both write to the same cache key, a data clobbering race condition emerges.

**Scenario:** `ProductService` owns price/description. `InventoryService` owns stock levels. Both invalidate and regenerate `cache:product:123` — a single blob containing both.

**The failure sequence:**

1. 12:00:00 — `ProductService` updates price from $100 to $80. Commits to DB. Invalidates `cache:product:123`.
2. 12:00:01 — A user request causes a cache miss. Application fetches from DB (price: $80, stock: 10) and writes the correct blob to `cache:product:123`.
3. 12:00:02 — `InventoryService` processes a sale, updates stock from 10 to 9. Commits to DB. To regenerate the cache, it reads the "full product" from a read replica. Due to replication lag, it reads the state from 12:00:00 and gets `{price: $100, stock: 10}`.
4. 12:00:03 — `InventoryService` updates its in-memory object to `{price: $100, stock: 9}` and writes this incorrect blob to `cache:product:123`.
5. 12:00:04 — All subsequent users see price: $100 (wrong) and stock: 9 (correct).

The `InventoryService` has clobbered the cache with stale price data because it was forced to read and write a blob containing data it doesn't own. This persists until the next invalidation.

---

### Single Key vs. Split Keys — The Trade-off

**Argument for a single cache key (blob approach):**
- Performance: One Redis round trip to get all data needed to render a page.
- Simplicity: Application logic is trivial — `GET cache:product:123`.
- Atomic(ish) view: When populated, the cache represents a single consistent snapshot.

**Argument for separate cache keys (granular approach):**
- Data ownership and consistency: `ProductService` only writes `cache:product:123:details`. `InventoryService` only writes `cache:product:123:inventory`. The clobbering race condition is eliminated by design.
- Independent invalidation: A description change only invalidates the details key. A stock change (which happens frequently) only invalidates the inventory key. Higher overall cache hit rate.
- Independent TTLs: Inventory is volatile — 10-second TTL. Product descriptions are stable — 24-hour TTL. A single blob forces you to use the shortest TTL for all data, which is inefficient.

**Recommendation: Split the cache keys.**

The single blob approach is a premature optimization that leads to severe data consistency bugs in a microservices environment. Showing an incorrect price or promising stock that doesn't exist is a critical business failure. The performance cost of an extra cache read is a solvable engineering problem. The consistency bug is a fundamental architectural flaw.

---

### Composing the Response Without Doubling Latency

The naive approach is two serial Redis calls — two network round trips. Use `MGET` instead.

**MGET (Multi-GET):**

```python
# One network round trip to Redis
data_array = redis.mget("cache:product:123:details", "cache:product:123:inventory")
details   = data_array[0]
inventory = data_array[1]
```

Redis processes this as a single command, retrieving all keys and returning them in one response. Network latency is effectively that of a single `GET` command.

**Pipelining (alternative):**

```python
pipe = redis.pipeline()
pipe.get("cache:product:123:details")
pipe.get("cache:product:123:inventory")
results = pipe.execute()  # One network round trip
```

Note: `MGET` is atomic at the command level. Pipelining is not. For reads this distinction doesn't matter, but it's important to know if writes were involved.

**Composition logic on cache miss:**

If any fragment returns `nil`, the application knows exactly which service's data to fetch. It calls the owning service (or its database), caches the result, and assembles the final response. The owning service boundary is preserved — the application layer never directly queries another service's database.

---

## 7. Multi-Region Caching — Consistency Across Geographies

When you have active deployments in multiple regions (e.g., us-east-1, ap-southeast-1, eu-west-1), each with its own Redis cluster, keeping regional caches consistent becomes a distributed systems problem.

### Option A: Invalidate on Write (Active Invalidation)

When a price is updated in the primary write region, actively send invalidation signals to all other regional Redis clusters.

**Real-world operational complexity:**

You cannot make synchronous API calls cross-region — the latency is high and unreliable. This requires a durable, asynchronous fan-out architecture:

1. `ProductService` in `us-east-1` publishes an invalidation event `{"productId": "123"}` to AWS SNS.
2. Each region has an SQS queue subscribed to this SNS topic.
3. Each region runs a dedicated "Invalidator" service that pulls from its local SQS queue and issues `DEL` commands to its local Redis cluster.

**Operational overhead this creates:**
- Monitor end-to-end invalidation pipeline latency (SNS publish → SQS delivery → worker poll → Redis DEL).
- Alert on SQS queue depth per region. A growing queue in `eu-west-1` means the European invalidator is down or lagging, and that entire region is serving stale data.
- Manage Dead-Letter Queues for messages that fail after multiple retries.
- Pay for SNS, SQS, and compute for invalidator workers in every region.

The end-to-end latency of this pipeline can still be several seconds. During this window, the system is in an inconsistent state.

### Option B: TTL-Based Expiry Only (Passive Expiration)

No cross-region invalidation infrastructure. Each regional cache simply expires on its TTL.

**Real-world operational complexity:**

- Radical simplicity: No cross-region messaging to build, manage, or monitor. Regional deployments are loosely coupled. A failure in EU Redis has zero impact on US.
- The complexity moves to the "source of truth" problem: when a cache miss occurs in Singapore, where does the application fetch fresh data? Calling back to `us-east-1` adds cross-region latency. A better solution is a global database technology like Amazon Aurora Global Database, which maintains read replicas in each region. The problem shifts to monitoring database replication lag.
- Business buy-in: The primary "complexity" is organizational, not technical. You must define acceptable staleness per data type with product and business teams. This becomes part of the SLA.

### Hybrid Recommendation

- For hyper-critical data (price, inventory): The business will not accept a 60-second TTL. Build the "Invalidate on Write" system, accept the complexity, and invest in making it robust.
- For less critical data (product descriptions, specifications, reviews): A 5–10 minute TTL is perfectly acceptable. Use TTL-based expiry to reduce complexity and cost.

---

### Option C: Immutable Versioned Cache Keys (Sidesteps Consistency Entirely)

Instead of mutating `cache:product:123` and invalidating it, version your cache keys: `cache:product:123:v47`.

The application always fetches the current version pointer from a lightweight, globally replicated config store (Cloudflare KV, DynamoDB Global Tables, or similar). When a price updates, you write a new version `cache:product:123:v48` and update the pointer. The old version expires naturally via TTL. The cached data itself never needs invalidation — it is immutable by design.

This is appropriate for data that changes infrequently but must be globally consistent when it does change. It is how large CDN-heavy systems handle surrogate key invalidation at their core.

---

### Regional Redis Cluster Total Loss — Recovery Sequence

**The scenario:** Singapore Redis cluster is completely destroyed (not a failover — a full cluster loss). CDN is still up.

**What users experience:**

The majority of users are completely unaffected. The CDN edge node in Singapore has cached copies of popular product pages with a 2-minute TTL. Requests are served directly from the CDN in ~50ms. The CDN acts as a powerful resilient shield, masking the Layer 3 failure entirely.

A small percentage of requests will be for long-tail products or will hit right after a TTL expiry. These are the "unlucky" users:

1. CDN forwards the request to origin in `ap-southeast-1`.
2. Application server attempts to connect to the Singapore Redis cluster. Connection times out.
3. The circuit breaker trips open. Subsequent requests fail fast.
4. Graceful degradation logic returns a `503 Service Unavailable` error page.

These users see an error page. All other users see a working site served by the CDN.

**The recovery sequence (on-call playbook):**

- T+0s (Alerting): PagerDuty fires. Key signals: 100% Redis connection errors, circuit breakers open, spike in origin 5xx responses, all localized to `ap-southeast-1`.
- T+5m (Triage): On-call engineer confirms total cluster loss (not a simple failover). CDN is correctly shielding most traffic. Critical incident, but not a full site-down catastrophe.
- T+10m (Initiate Recovery): Use IaC scripts (Terraform/CloudFormation) to provision a new ElastiCache for Redis cluster in `ap-southeast-1`. Pre-tested, one-command action.
- T+25m (New Cluster Ready): New, empty cluster is available. Do NOT point production traffic at it yet.
- T+30m (Cache Warming — Critical Step): Run an automated cache warmer script:
  - Fetch the top N (e.g., 20,000) most frequently accessed product IDs from analytics or access logs.
  - Iterate through these IDs, making requests to the application (configured to use the new Redis cluster) to trigger a cache fill.
  - This pre-populates the cache with controlled, gradual load on the database instead of a thundering herd of real users.
- T+40m (Phased Switchover): Use a feature flag or DNS weighting to slowly shift production traffic to application servers configured with the new, warmed cache. Monitor DB CPU and cache hit rates closely.
- T+45m (Full Recovery): All traffic on the new cluster. Incident resolved. Post-mortem focuses on root cause and automating the recovery sequence further.

---

## 8. Diagnosing Cache-Related Production Issues

The hardest production problems are when all infrastructure monitors are green but a business metric (e.g., conversion rate) has dropped. The diagnostic approach is to work backwards from user-visible impact, moving systematically down the stack.

### Step 1: Frame the Problem at the Business Layer

Before looking at a single infrastructure graph, understand the nature of the drop. Conversion is a funnel. Where in the funnel is the drop?

**Metrics to check first:**
- Conversion funnel breakdown: View Product Page → Add to Cart → Start Checkout → Complete Purchase.
  - Drop between View Product → Add to Cart: Problem is on the Product Detail Page. Prime suspects: incorrect price/inventory (stale cache) or slow page load (ineffective cache).
  - Drop between Add to Cart → Start Checkout: Problem likely with cart or session management. Points to user-specific data in Redis.
- Real User Monitoring (RUM) data (Datadog RUM, New Relic Browser): Core Web Vitals (LCP, FID, CLS) and JS error rates, segmented by page type.
  - Spike in LCP on PDPs: Smoking gun for performance degradation. Main content is taking longer to appear. Points directly to slow origin response, likely due to cache miss.
  - Spike in JS errors: Application may be sending malformed data (e.g., broken JSON from a corrupted cache entry), causing the "Add to Cart" button to fail silently.

### Step 2: Investigate the Edge (CDN Dashboard)

**Hypothesis:** Origin servers are slower to respond, and the CDN is either causing it or correctly reporting it.

- Cache Hit Ratio (time-series): A significant drop (e.g., from 80% to 60%) means the CDN is sending much more traffic to origin. Could be a misconfiguration in a recent deployment (changed cache-control headers) or mass invalidations.
- Origin Latency (p95/p99): Time for origin to respond from the CDN's perspective. If this is up by ~500ms, the problem is within the data center. The CDN is just the messenger.

### Step 3: Investigate the Application and Cache (APM + Redis Dashboard)

**Hypothesis:** The application is spending more time waiting for Redis or the database.

- Application endpoint latency breakdown (APM trace): A trace of a slow `/api/products/{id}` request showing time spent in each span. Expect to see a large increase in time spent in `redis.mget` or `db.query`.
- Redis Hit Ratio: If the CDN hit ratio was stable but the Redis hit ratio has dropped, there is an internal cache-churning problem. Requests are getting past the CDN but missing in the primary cache, forcing fallback to the DB.
- Redis Evictions and Memory Usage: A spike in evictions is a massive red flag. It means Redis is running out of memory and is being forced to throw out keys. If memory usage hit its ceiling over the weekend and evictions began to climb, this is likely the root cause.

  Also check `redis-cli info stats` for `keyspace_hits` vs `keyspace_misses` as a direct ratio, and `info memory` for `mem_fragmentation_ratio`. A fragmentation ratio above 1.5 means Redis is using significantly more physical memory than it thinks it is, which can cause unexpected evictions even when reported memory usage looks fine.

### Step 4: Confirm at the Database Layer

- DB CPU and active connections: Confirms the impact of lower Redis hit rate. If Redis evictions are up, expect DB CPU to climb from baseline 40% to 80–90%.
- Read replica replication lag: Check for the stale data theory. If a bug caused mass invalidations and DB replicas were lagging, the application may have re-cached old pre-sale prices.

### Tying It Together — A Realistic Root Cause Story

"On Friday evening, a deployment for the new 'Social Wishlist' feature went out. This feature caches large user-specific objects in the primary Redis cluster. Over the weekend, as usage grew, Redis memory filled up, causing the `maxmemory-policy` (e.g., `allkeys-lru`) to start aggressively evicting keys. The least recently used keys were often long-tail product data.

This dropped the effective L3 cache hit rate from 95% to 70%. As a result, 25% more PDP traffic had to fall back to the database. This increased the p95 latency of the product API from 150ms to 650ms, which increased page load times. This added friction led directly to a 3% drop in the Add to Cart conversion rate."

---

## 9. Key Concepts Quick Reference

| Concept | What It Is | When to Use |
|---|---|---|
| Cache Busting | Embed content hash in filename (e.g., `app.a1b2c3d4.js`) | Versioned static assets with long TTLs |
| Cache Tags / Surrogate Keys | Tag cached content; purge by tag not URL | Precise invalidation of related content across CDN |
| Cache Warming | Pre-populate cache before routing traffic | After cache loss, before new cluster goes live |
| Request Collapsing | CDN/proxy holds N identical requests, sends 1 to origin | Preventing thundering herd on cache miss |
| Probabilistic Early Expiration (XFetch) | Randomly refresh cache before hard expiry | High-traffic keys where stampede is a risk |
| Circuit Breaker | Fail fast when dependency is down | Redis or DB outage — prevent cascading failure |
| MGET / Pipelining | Fetch multiple Redis keys in one round trip | Split cache keys that must be composed |
| CDC (Change Data Capture) | Read DB transaction log to trigger invalidation | Decoupling invalidation from application code |
| Immutable Versioned Keys | Version the key, update a pointer | Sidestep invalidation entirely for infrequently changing data |
| `allkeys-lru` eviction policy | Redis evicts least recently used keys when memory is full | Default for general caching workloads — monitor eviction rate |
| `mem_fragmentation_ratio` | Physical memory / logical memory in Redis | >1.5 indicates fragmentation causing unexpected evictions |

---