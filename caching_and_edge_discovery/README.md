# Caching & Edge Delivery — Distributed Cache + CDN

A deep-dive into designing a production-grade multi-layer caching and edge delivery system for a global e-commerce platform — covering cache decision frameworks, layer-by-layer architecture, invalidation strategies, stampede prevention, failure math, key design, multi-region consistency, and operational incident analysis.

---

## Table of Contents

1. [Problem Context](#problem-context)
2. [Requirements & Constraints](#requirements--constraints)
3. [What to Cache — The Decision Framework](#what-to-cache--the-decision-framework)
4. [The Multi-Layer Caching Architecture](#the-multi-layer-caching-architecture)
5. [Cache Invalidation in Practice — The Flash Sale Scenario](#cache-invalidation-in-practice--the-flash-sale-scenario)
6. [Cache Stampede — The Thundering Herd Problem](#cache-stampede--the-thundering-herd-problem)
7. [Redis Failover — The Math of Cache Dependency](#redis-failover--the-math-of-cache-dependency)
8. [Cache Key Design and Data Ownership](#cache-key-design-and-data-ownership)
9. [Multi-Region Caching — Consistency Across Geographies](#multi-region-caching--consistency-across-geographies)
10. [Observability & Alerting](#observability--alerting)
11. [Production Incident Analysis](#production-incident-analysis)
12. [Key Concepts Quick Reference](#key-concepts-quick-reference)

---

## Problem Context

A growing e-commerce platform serving 10 million daily active users is experiencing:

- High latency on product detail pages (avg 800ms)
- Database CPU spiking to 90%+ during flash sales
- Users in Southeast Asia and Europe complaining about slow load times (origin servers in US-East)
- Occasional cache stampedes bringing down the DB

The platform has:
- Product catalog pages, images, and user-specific recommendations
- Flash sales with 500 products simultaneously dropping in price
- Multiple microservices: `ProductService` (owns price/description), `InventoryService` (owns stock levels)
- Multi-region expansion to `ap-southeast-1` and `eu-west-1`

---

## Requirements & Constraints

### Functional
- Product detail pages must load in < 100ms for users in Singapore and Europe
- Flash sale price updates must be visible to all users within 30 seconds
- User-specific recommendations must be personalized and fast
- Cache must survive a Redis primary failure without taking down the database

### Non-Functional

| Requirement | Target | Rationale |
|---|---|---|
| CDN cache hit ratio | > 95% for product pages | Offload origin traffic |
| Redis cache hit ratio | > 95% for application layer | Protect database |
| DB load during Redis failover | Must not exceed 5,000 QPS | DB capacity limit |
| Flash sale invalidation SLA | < 30 seconds globally | Business requirement |
| CDN purge propagation | 2–5s (Cloudflare/Fastly), 10–30s (CloudFront) | Provider-dependent |

### The Core Insight

Caching is not a single layer — it is a defense-in-depth strategy. Each layer absorbs a specific type of load and protects the layer beneath it. A request that can be served from the browser cache never touches the CDN. A request served from the CDN never touches the origin. A request served from Redis never touches the database.

---

## What to Cache — The Decision Framework

Before architecting a solution, evaluate every piece of data against four axes.

### The Four Axes

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CACHING DECISION FRAMEWORK                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Axis 1: Read/Write Ratio — The Golden Rule                            │
│  ─────────────────────────────────────────                              │
│  High Read / Low Write  →  Excellent candidate                         │
│  Low Read / High Write  →  Poor candidate                              │
│                                                                         │
│  Axis 2: Cost of Generation — "Bang for Your Buck"                     │
│  ─────────────────────────────────────────────────                      │
│  High Cost (ML model, 5-table join)  →  Strong candidate               │
│  Low Cost (primary key lookup)       →  Weak candidate                 │
│                                                                         │
│  Axis 3: Staleness Tolerance — "How Fresh is Fresh Enough?"            │
│  ──────────────────────────────────────────────────────                 │
│  High tolerance (descriptions, images)  →  Long TTL, CDN-friendly     │
│  Low tolerance (price, inventory)       →  Short TTL, careful design  │
│  Zero tolerance (payment status)        →  Never cache                 │
│                                                                         │
│  Axis 4: Scope — "Who Is This For?"                                    │
│  ──────────────────────────────────                                     │
│  Global/Public (same for all users)  →  CDN edge caching              │
│  User-Specific/Private               →  Redis keyed by user_id        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### The Decision Matrix

| Data Type | Read/Write | Cost to Generate | Staleness Tolerance | Scope | Verdict |
|---|---|---|---|---|---|
| Product Images, JS/CSS | Very High Read | Low | Very High | Global | Excellent. CDN with long TTL + cache busting. |
| Product Details (Desc, Specs) | Very High Read | Medium (DB join) | High | Global | Excellent. CDN + central cache. |
| Product Price | Very High Read | Low | Low | Global | Good, short TTL (1–5 min). Business must define acceptable staleness. |
| Inventory Count | High Read | Low | Very Low | Global | Tricky. Very short TTL (10s) or cache status ("high/low/out") instead of exact count. Final check always at checkout. |
| User Recommendations | High Read | Very High (ML/DB) | Medium | User-Specific | Excellent. Redis keyed by user_id. |
| Shopping Cart | Low Read / High Write | Low | Zero | User-Specific | Poor. Generally should not be cached. |
| "Top 10" Lists | Very High Read | High (Aggregation) | Medium | Global | Excellent. Central cache, regenerate on schedule. |
| Payment Transaction Status | High Read | Low | Zero | User-Specific | Never cache. Must be real-time. |

---

## The Multi-Layer Caching Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    MULTI-LAYER CACHING ARCHITECTURE                     │
│                                                                         │
│   User's Device (Singapore)                                             │
│   ┌──────────────────────────────────────────────────────────────────┐ │
│   │  Layer 1: Browser Cache                                          │ │
│   │  • Static assets (CSS/JS/fonts): 1 year TTL + cache busting     │ │
│   │  • Product images: 24 hours TTL                                  │ │
│   │  • Cache-Control headers + ETag validation                       │ │
│   └──────────────────────────────────────────────────────────────────┘ │
│                    │ Cache Miss                                          │
│                    ▼                                                     │
│   Internet → CDN Edge Node (Singapore PoP)                              │
│   ┌──────────────────────────────────────────────────────────────────┐ │
│   │  Layer 2: CDN Edge Cache                                         │ │
│   │  • Static assets: 1 year TTL                                     │ │
│   │  • Product images: 7–30 days TTL                                 │ │
│   │  • Product detail pages + API responses: 2 min TTL              │ │
│   │  • Cache Tags for targeted invalidation                          │ │
│   │  • Request Collapsing (Cloudflare Cache Lock / Fastly)           │ │
│   └──────────────────────────────────────────────────────────────────┘ │
│                    │ Cache Miss                                          │
│                    ▼                                                     │
│   Origin Data Center (US-East)                                          │
│   ┌──────────────────────────────────────────────────────────────────┐ │
│   │  Layer 3: Application-Level Cache (Redis)                        │ │
│   │  • User recommendations: 1–4 hours, keyed by user_id            │ │
│   │  • "Top 10" lists: 20 min TTL, regenerated every 15 min         │ │
│   │  • DB query results: 5–10 min TTL                                │ │
│   │  • Session data                                                   │ │
│   │  • Multi-AZ ElastiCache with automatic failover                  │ │
│   └──────────────────────────────────────────────────────────────────┘ │
│                    │ Cache Miss                                          │
│                    ▼                                                     │
│   ┌──────────────────────────────────────────────────────────────────┐ │
│   │  Layer 4: Database Buffer Pool                                   │ │
│   │  • Frequently accessed data blocks from disk                     │ │
│   │  • Query execution plans                                         │ │
│   │  • Managed by DB engine (InnoDB Buffer Pool / pg shared_buffers) │ │
│   │  • Allocate 70–80% of server RAM                                 │ │
│   └──────────────────────────────────────────────────────────────────┘ │
│                    │ Buffer Pool Miss                                    │
│                    ▼                                                     │
│   Database Disk                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Layer 1: Browser Cache

**What it caches:** Static assets (CSS, JS, fonts, site-wide images), product images after first view.

**Technology:** Standard HTTP caching headers.
- `Cache-Control: public, max-age=31536000, immutable` — for versioned/hashed assets.
- `ETag` — for validation. Browser sends its ETag; server responds `304 Not Modified` if unchanged, saving bandwidth.

**TTL Strategy — Cache Busting:**

Versioned assets use content-hash filenames (e.g., `app.a1b2c3d4.js`). When the file changes, the name changes. This allows a 1-year `max-age`. Users always get the new version on deploy, and cache it indefinitely otherwise.

```
Build process:
  app.js → app.a1b2c3d4.js  (hash of file content)
  app.css → app.f9e8d7c6.css

HTML references:
  <script src="/static/app.a1b2c3d4.js"></script>

On deploy:
  New file: app.b2c3d4e5.js  (different hash)
  Old file: app.a1b2c3d4.js  (still cached in browsers — that's fine, it's the old version)
  HTML updated to reference new hash → users get new version
```

**Failure Mode:** Extremely resilient. If the user clears their cache, the browser simply makes a new request to the CDN. No system-wide impact, just a slightly slower first load for that user.

---

### Layer 2: CDN (Content Delivery Network)

The most critical layer for solving global latency (EU/SEA users) and offloading origin traffic.

**Technology selection matters:**

| Provider | Purge Propagation | Best For |
|---|---|---|
| Cloudflare | 2–5 seconds globally | Tight invalidation SLAs, Cache Tags |
| Fastly | 2–5 seconds globally | Cache Tags, Request Collapsing |
| AWS CloudFront | 10–30 seconds globally | AWS-native integrations |

For a 30-second flash sale SLA, Cloudflare or Fastly is the right choice. CloudFront's 10–30 second propagation leaves very little margin.

**TTL Strategy:**

| Content Type | TTL | Invalidation Method |
|---|---|---|
| Static assets (JS/CSS) | 1 year | Cache busting (filename hash) |
| Product images | 7–30 days | CDN API purge on update |
| Product detail pages | 2 minutes | Cache Tags purge on price/inventory change |
| API responses (`GET /api/products/{id}`) | 2 minutes | Cache Tags purge |

**Cache Tags (Surrogate Keys):**

Tag the cached page for product 123 with `product-123`. When the backend updates that product's price, one API call purges all content tagged `product-123` across all PoPs globally. More precise and efficient than URL-based purges.

```
Response header from origin:
  Cache-Tag: product-123, category-shoes, brand-nike

Purge request:
  POST /api/v4/purge
  {"tags": ["product-123"]}

Effect: All cached content tagged "product-123" is invalidated globally
```

**Failure Mode:** A full CDN outage is a Major Incident. All traffic routes directly to origin, immediately triggering DB CPU spikes and likely a full site outage. Mitigation: have a DNS-based failover plan, and ensure origin infrastructure has robust rate limiting and load shedding to survive the initial flood.

---

### Layer 3: Application-Level Cache (Redis)

**Technology:** Redis with rich data structures — user profiles in a Hash, "recently viewed items" in a Sorted Set. Run as AWS ElastiCache for Redis with Multi-AZ configuration for automatic failover.

**TTL Strategy:**

| Data | TTL | Key Pattern |
|---|---|---|
| User recommendations | 1–4 hours | `cache:recommendations:user:{user_id}` |
| "Top 10" lists | 20 minutes | `cache:top10:bestsellers` |
| DB query results | 5–10 minutes | `cache:product:{product_id}:details` |
| Session data | 30 minutes (sliding) | `session:{session_id}` |

**Cache Warming Strategy for "Top 10" Lists:**

A background job runs every 15 minutes, queries the database, and writes the result to Redis with a 20-minute TTL. This ensures there is never a gap where the key is expired and a stampede hits the DB.

```
Background job (every 15 min):
  result = db.query("SELECT * FROM products ORDER BY sales DESC LIMIT 10")
  redis.set("cache:top10:bestsellers", serialize(result), ttl: 1200)  # 20 min
```

**Failure Mode:** A Redis outage is a Critical Incident. All requests that relied on the cache now hit the database directly, causing the exact CPU spike problem the cache was designed to prevent. Mitigation: Multi-AZ provides automatic failover (30–60 second window). Application code must use a circuit breaker — if the cache is down, fail fast or degrade gracefully rather than falling through to the DB for every request.

---

### Layer 4: Database Buffer Pool

**Technology:** Internal to the database engine (InnoDB Buffer Pool in MySQL, `shared_buffers` in PostgreSQL).

**Tuning:** Allocate 70–80% of server RAM to the buffer pool. Monitor the buffer pool hit ratio as a key operational metric. An undersized buffer pool forces constant disk reads, resulting in high I/O wait and slow queries.

**Failure Mode:** This doesn't "fail" in the traditional sense. The failure mode is an undersized or poorly configured cache. The failure is silent — queries just get slower as the hit rate drops.

---

## Cache Invalidation in Practice — The Flash Sale Scenario

Cache invalidation is one of the hardest problems in distributed systems. The flash sale scenario makes it concrete: 500 products drop in price simultaneously at 12:00 PM, and users must see the new prices within 30 seconds.

### Pre-Scheduling (The Right Approach)

Price updates are not triggered by a human clicking a button at noon. They are pre-scheduled and loaded into a durable message queue (AWS SQS). Each message contains `product_id`, `new_price`, and `sale_start_time`. A pool of workers begins processing this queue at exactly 12:00:00 PM. This decouples the trigger from the execution and provides reliability via retry semantics.

### The End-to-End Invalidation Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    FLASH SALE INVALIDATION TIMELINE                     │
│                                                                         │
│  T+0s to T+5s — DB Write + Cache Invalidation                         │
│  ─────────────────────────────────────────────                          │
│  20 workers pull from SQS queue in batches                             │
│                                                                         │
│  For each product (e.g., product-123):                                 │
│  1. UPDATE products SET price = :new_price WHERE id = 123  ← commit   │
│  2. redis.del("cache:product:123:details")                 ← L3 purge  │
│  3. Collect tag "product-123" into batch list              ← don't call CDN yet │
│  4. When batch reaches 25 products (or 1s timeout):                    │
│     POST /api/v4/purge {"tags": ["product-123", "product-124", ...]}  │
│     (One API call for 25 products, not 25 separate calls)             │
│                                                                         │
│  T+5s to T+10s — Global CDN Propagation                               │
│  ─────────────────────────────────────────                              │
│  CDN control plane propagates invalidation to all PoPs                 │
│  Cloudflare/Fastly: 2–5 seconds                                        │
│  CloudFront: 10–30 seconds                                             │
│  Singapore PoP: cache entry for product-123 marked invalid             │
│                                                                         │
│  T+10s — User Request in Singapore                                     │
│  ─────────────────────────────────                                      │
│  User requests product-123 page                                        │
│  CDN PoP: cache miss (purged)                                          │
│  PoP forwards request to origin over CDN backbone network              │
│                                                                         │
│  T+10s to T+10.3s — Origin Fetch                                      │
│  ────────────────────────────────                                       │
│  Application checks Redis: miss (invalidated)                          │
│  Application queries database: gets new price ($80)                    │
│  Renders full HTML page                                                 │
│  Response includes: Cache-Tag: product-123                             │
│                                                                         │
│  T+10.3s to T+10.5s — Re-caching and Delivery                        │
│  ─────────────────────────────────────────────                          │
│  Singapore PoP stores fresh copy with product-123 tag and 2-min TTL   │
│  Streams response to user                                              │
│                                                                         │
│  RESULT: User sees correct flash sale price ~10–15 seconds after noon  │
│          Well within the 30-second SLA                                 │
└─────────────────────────────────────────────────────────────────────────┘
```

### Failure Points and Mitigations

**Failure 1: DB Overloaded During Bulk Updates**

| Problem | Mitigation |
|---|---|
| 20 workers processing 500 products simultaneously could overload the database | Queue throttling (control worker concurrency), connection pooling, idempotent retry logic. SQS visibility timeout ensures failed messages are retried. DLQ catches messages that fail repeatedly for manual inspection. |

**Failure 2: CDN Purge API Fails**

| Problem | Mitigation |
|---|---|
| DB is updated, but the CDN API returns a 5xx. CDN continues serving stale pre-sale prices. | Retry with exponential backoff. If purge fails after 3–4 retries, log failed tags to PagerDuty AND write them to a separate retry queue. The 2-minute TTL is the ultimate safety net — stale content expires automatically within 2 minutes even if purge fails completely. |

**Failure 3: The "Split Brain" — DB Write Succeeds, Invalidation Never Fires**

This is the most insidious failure. The worker commits the DB transaction, then the process crashes before issuing the CDN or Redis invalidation. The DB is correct but the cache is permanently stale until TTL expiry.

**Mitigation — Change Data Capture (CDC):**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CDC-BASED INVALIDATION ARCHITECTURE                  │
│                                                                         │
│  Application                                                            │
│       │                                                                 │
│       ▼                                                                 │
│  Database (US-East)                                                     │
│       │                                                                 │
│       ▼ (WAL / transaction log)                                         │
│  Debezium (CDC connector)                                               │
│  Reads database transaction log                                         │
│  Detects change in products.price column                               │
│       │                                                                 │
│       ▼                                                                 │
│  Kafka Topic: "product-price-changes"                                  │
│       │                                                                 │
│       ▼                                                                 │
│  Invalidator Service (dedicated, highly-available)                     │
│  Only job: consume events → issue CDN purge + Redis DEL                │
│                                                                         │
│  WHY THIS IS BETTER:                                                    │
│  Application's only responsibility is writing to the database.         │
│  Any committed change will eventually trigger an invalidation event,   │
│  even if the main application crashes.                                 │
│  Moves from "at-least-once" application retries to a reliable,        │
│  event-driven architecture.                                            │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Cache Stampede — The Thundering Herd Problem

### What Is a Cache Stampede?

When a popular cache key expires (or is purged), many concurrent requests simultaneously find a cache miss and all attempt to regenerate the data from the database at the same time. This creates a sudden spike of expensive DB queries.

### The Naive Fix: SETNX Mutex — Why It Fails at Scale

The `SETNX` (SET if Not eXists) approach acquires a lock so only one process regenerates the data while others wait. This works at small scale but catastrophically fails under a true thundering herd.

**The failure scenario with 50,000 concurrent users hitting a cold cache:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SETNX MUTEX FAILURE ANALYSIS                         │
│                                                                         │
│  CDN PoP: cache for Product X is empty (just purged)                   │
│  No request collapsing configured                                       │
│  50,000 requests forwarded to origin simultaneously                    │
│                                                                         │
│  Application servers: 10 servers × 100 workers = 1,000 total capacity  │
│  First 1,000 requests accepted. Other 49,000 queue in TCP listen buffers│
│                                                                         │
│  Worker A: SETNX product:X:lock → SUCCESS → proceeds to DB (~200ms)   │
│  Workers B-1000: SETNX product:X:lock → FAIL → enter spin-wait loop:  │
│    sleep(50ms) → check lock → sleep(50ms) → check lock → ...          │
│                                                                         │
│  CRITICAL: These 999 workers are FULLY OCCUPIED.                       │
│  They cannot serve any other traffic.                                  │
│                                                                         │
│  All 1,000 workers are now tied up:                                    │
│    1 worker: talking to DB                                             │
│    999 workers: spin-waiting on a lock                                 │
│                                                                         │
│  New requests (Product Y, checkout, health checks): REJECTED           │
│  Load balancer health checks: FAIL                                     │
│  App servers marked unhealthy, removed from rotation                   │
│                                                                         │
│  RESULT: Self-inflicted Denial of Service.                             │
│  Database is protected. Application tier is sacrificed.                │
│  Connection pool starvation.                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

**The correct fix is at the layer before the application servers:**

| Solution | How It Works | Provider |
|---|---|---|
| CDN Request Collapsing (best) | CDN holds 49,999 requests at the edge, sends 1 to origin, fans response out to all waiting clients. Application never sees the stampede. | Cloudflare "Cache Lock", Fastly "Request Collapsing", Varnish "Grace Mode" |
| API Gateway / Proxy Layer | NGINX or Envoy configured to perform request collapsing before requests reach application code. | NGINX, Envoy |

---

### The Production-Grade Fix: Probabilistic Early Expiration (XFetch Algorithm)

This pattern eliminates the "hard expiration" event that creates the vulnerability window in the first place. Based on the XFetch algorithm (Vattani et al., 2015).

**How it works mechanically:**

Instead of storing just the value, store a richer object in Redis:

```json
{
  "data": "<html>...</html>",
  "created_at": 1678881600,
  "ttl": 120
}
```

On every request for Product X:

```
1. Fetch the full object from Redis.

2. Immediately serve object.data to the user.
   User gets a fast response. Data may be slightly stale, but served instantly.

3. After serving, check whether this request should trigger a background refresh:

   FORMULA (XFetch algorithm):
   current_time - (delta * log(rand())) >= created_at + ttl

   Where:
   - delta = configurable window (e.g., 10 seconds) — how early to start refreshing
   - rand() = random float between 0.0 and 1.0
   - log(rand()) = value from -infinity to 0
   - -(delta * log(rand())) = random positive number with exponential distribution
     (small values far more likely than large ones)

4. As current_time approaches created_at + ttl, the probability of satisfying
   the condition increases exponentially. One request is eventually "chosen."

5. Chosen request:
   SETNX product:X:refresh_lock 1 EX 10  (short-lived refresh lock)
   If lock acquired: re-fetch from DB, write new object with updated created_at
   If lock not acquired: another request was chosen moments before — do nothing
```

**Why this is better than the mutex:**

| Dimension | SETNX Mutex | Probabilistic Early Expiration |
|---|---|---|
| Hard expiration event | Yes — creates a vulnerability window | No — cache is continuously refreshed before expiry |
| User latency during miss | All 50,000 users wait | Only one user triggers a background refresh; all others get stale data instantly |
| DB load pattern | Sharp spike every TTL interval | Low, constant hum of single background refreshes |
| Application server impact | Connection pool starvation | No impact — workers are never blocked waiting |
| Complexity | Simple to implement | Requires storing metadata alongside cached value |

---

## Redis Failover — The Math of Cache Dependency

### The Scenario

Redis primary dies. ElastiCache promotes a replica. Failover takes 30–60 seconds. Application is configured to "fail open" — cache misses fall through to the database.

Platform: 500K concurrent users at 2:00 AM. Normal cache hit rate: 95%. Database capacity: 5,000 QPS before degradation.

### The Math

```
Step 1: Calculate total platform RPS
  500K concurrent users, each performing a cache-dependent action every ~10 seconds
  Total RPS = 500,000 / 10 = 50,000 RPS

Step 2: Normal database load (baseline)
  Normal DB Load = 50,000 RPS × 5% miss rate = 2,500 QPS
  Healthy — 50% of the 5,000 QPS capacity, with a good buffer.

Step 3: Database load during Redis failover
  Cache hit rate drops to 0%. All 95% of requests fall through to DB.
  New load from cache misses = 50,000 × 95% = 47,500 QPS
  Total DB load = 2,500 + 47,500 = 50,000 QPS

Step 4: The verdict
  50,000 QPS / 5,000 QPS capacity = 10x overload

  Database instantly exhausts its connection pool.
  Queries time out. CPU spikes to 100%.
  Application servers waiting on DB connections exhaust their own worker pools.
  The site goes down.
```

**"Fail open to DB" is a naive and dangerous strategy for a critical dependency like a cache.**

### Mitigation 1: Circuit Breaker

```
States:
  CLOSED (normal):
    Requests pass through to Redis.
    Monitor for failure threshold (e.g., >50% timeouts over a 5-second window).

  OPEN (failure):
    Threshold breached. Breaker trips.
    For the next 60 seconds, any call to Redis fails fast WITHOUT attempting a connection.
    Application does NOT fall back to the database. Instead:
      - Non-essential data (recommendations): Return empty set. Page renders without component.
      - Essential data (product page): Return 503 Service Unavailable with friendly error page.
    This sheds load instantly and protects the system.

  HALF-OPEN (recovery):
    After 60 seconds, allow one test request.
    If it succeeds (failover complete): close the breaker.
    If it fails: stay open.
```

### Mitigation 2: Read from Replica (Architecturally Superior)

```
Configuration:
  Redis client aware of both primary endpoint and reader endpoint.
  AWS ElastiCache Reader Endpoint: load balances across available replicas.

Normal operation:
  Write to primary.
  Read from primary for strongest consistency.

During failover (primary unavailable):
  Read from primary fails or times out.
  Immediately retry against Reader Endpoint.
  Use very short connection timeout (50–100ms) — don't wait for full request timeout.

Why this is better:
  During the 60-second failover, replicas are still up with data milliseconds stale.
  For 99% of use cases (product info, categories), slightly stale data is infinitely
  better than returning an error.
  Replicas continue to absorb the 47,500 RPS of read traffic.
  Database never sees the spike.
```

**Combined strategy:** Use "Read from Replica" as the primary defense. Layer a circuit breaker on the entire Redis interaction (primary + replicas) as a final backstop for a total cluster-wide outage.

---

## Cache Key Design and Data Ownership

### The Race Condition Problem

When two services own different parts of the same cached object, you get a silent data corruption bug.

**The scenario:** `ProductService` owns price and description. `InventoryService` owns stock levels. Both write to the same Redis key `cache:product:123:details`.

```
Timeline:
  T+0ms:   User requests product 123. Cache miss. Both services start fetching.

  T+50ms:  InventoryService finishes first.
           Writes: { product_id: 123, stock: 42, price: null, description: null }
           redis.set("cache:product:123:details", {...}, ttl: 300)

  T+80ms:  ProductService finishes.
           Writes: { product_id: 123, stock: null, price: "$99", description: "..." }
           redis.set("cache:product:123:details", {...}, ttl: 300)

  RESULT:  Cache contains ProductService's version.
           InventoryService's stock update is silently overwritten.
           Next 300 seconds: all users see stale stock data.
           No error thrown. No alert fired. Silent corruption.
```

### Solution: Split Keys by Ownership Boundary

Each service owns exactly one cache key. No service ever writes to another service's key.

```
cache:product:{id}:details      ← owned exclusively by ProductService
cache:product:{id}:inventory    ← owned exclusively by InventoryService
cache:product:{id}:ratings      ← owned exclusively by RatingsService
```

**Composition on cache miss (Python example):**

```python
def get_product_page(product_id: str) -> dict:
    # Fetch all keys in a single round-trip using pipelining
    pipe = redis.pipeline()
    pipe.get(f"cache:product:{product_id}:details")
    pipe.get(f"cache:product:{product_id}:inventory")
    pipe.get(f"cache:product:{product_id}:ratings")
    details_raw, inventory_raw, ratings_raw = pipe.execute()

    # Determine what needs to be fetched from source
    details   = json.loads(details_raw)   if details_raw   else fetch_and_cache_details(product_id)
    inventory = json.loads(inventory_raw) if inventory_raw else fetch_and_cache_inventory(product_id)
    ratings   = json.loads(ratings_raw)   if ratings_raw   else fetch_and_cache_ratings(product_id)

    # Compose the full response in the application layer
    return {**details, **inventory, **ratings}

def fetch_and_cache_details(product_id: str) -> dict:
    data = product_service.get(product_id)
    redis.set(f"cache:product:{product_id}:details", json.dumps(data), ex=300)
    return data
```

**Why `pipeline()` matters:** A single `MGET` or pipeline call fetches all three keys in one network round-trip (~0.5ms) instead of three sequential calls (~1.5ms). At 50,000 RPS, this difference is significant.

### Single Key vs Split Keys — Trade-off Table

| Dimension | Single Shared Key | Split Keys by Owner |
|---|---|---|
| Network round-trips | 1 GET | 1 pipeline (still 1 round-trip) |
| Race condition risk | High — last writer wins | None — each service owns its key |
| Partial invalidation | Impossible — must invalidate everything | Precise — invalidate only what changed |
| Cache hit rate | Lower — any component change invalidates all | Higher — independent TTLs per component |
| Composition logic | In cache layer | In application layer (explicit, testable) |
| Operational clarity | Unclear ownership | Clear — grep the key prefix to find the owner |

**Rule of thumb:** If two different services need to write to the same cache key, that is a design smell. Split the key.

---

## Multi-Region Caching — Consistency Across Geographies

The platform expands to `ap-southeast-1` (Singapore) and `eu-west-1` (Ireland). Each region has its own Redis cluster. The question is: when a product price changes in US-East, how do the Singapore and Dublin Redis clusters get invalidated?

### Option A: Invalidate on Write (Active Invalidation)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ACTIVE INVALIDATION ARCHITECTURE                     │
│                                                                         │
│  US-East (Primary)                                                      │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  ProductService writes new price to DB                           │  │
│  │  Publishes event to SNS Topic: "product-price-changes"           │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                    │                                                     │
│         ┌──────────┴──────────┐                                         │
│         ▼                     ▼                                         │
│  SQS Queue (AP)         SQS Queue (EU)                                  │
│         │                     │                                         │
│         ▼                     ▼                                         │
│  Invalidator (SG)       Invalidator (EU)                                │
│  redis.del(key)         redis.del(key)                                  │
│                                                                         │
│  Latency: ~100–200ms cross-region (SNS → SQS → consumer)              │
│  Guarantee: At-least-once delivery (SQS retry semantics)               │
└─────────────────────────────────────────────────────────────────────────┘
```

**Operational overhead:** You now have two additional SQS queues, two additional consumer services, and cross-region SNS fan-out to operate and monitor. Every new region adds another queue + consumer pair.

### Option B: TTL-Based Expiry Only (Passive Invalidation)

No cross-region messaging. Each regional Redis cluster has a short TTL (e.g., 60 seconds). Stale data expires naturally.

**Trade-off:** Simple to operate. Accepts up to 60 seconds of staleness. For most product data, this is acceptable. For flash sale prices, it may violate the 30-second SLA.

### Option C: Immutable Versioned Cache Keys (The "Sidestep" Pattern)

Instead of invalidating, sidestep the problem entirely by making cache keys version-aware.

```
# Old approach — requires invalidation
cache:product:123:details  ← must be deleted when price changes

# Versioned approach — no invalidation needed
cache:product:123:v{version}:details

# Example:
cache:product:123:v42:details   ← current version
cache:product:123:v43:details   ← new version after price change
```

**How it works:**
1. A global version store (DynamoDB Global Tables or similar) maps `product_id → current_version`.
2. On every cache read, the application first fetches the current version number, then constructs the key.
3. When a price changes, the version is incremented. Old keys are simply never read again — they expire via TTL.
4. No cross-region invalidation messages needed. The version store is the source of truth.

**Trade-off:** Adds one extra lookup (version fetch) per cache read. Mitigate by caching the version number itself with a very short TTL (5–10 seconds). Accepts up to 5–10 seconds of staleness (the version cache TTL).

### Recommendation: Hybrid

| Data Type | Strategy | Rationale |
|---|---|---|
| Flash sale prices | Active invalidation (Option A) | Must meet 30s SLA |
| Product descriptions, images | TTL-only (Option B) | 60s staleness acceptable, zero operational overhead |
| Static assets (JS/CSS) | Immutable versioned keys (Option C) | Never need invalidation, infinite TTL |

### Regional Redis Cluster Total Loss — Recovery Sequence

A regional Redis cluster in Singapore is completely lost (AZ failure, data corruption). On-call playbook:

```
T+0m   — Alert fires: Redis connection errors spike in ap-southeast-1.
          On-call acknowledges. Confirms cluster is unresponsive.

T+2m   — Immediate action: Enable circuit breaker for Singapore Redis.
          Application falls back to: serve stale CDN content where possible,
          return 503 for user-specific data (recommendations, cart).
          DB is NOT used as fallback — circuit breaker prevents the 10x overload.

T+5m   — Assess recovery options:
          Option 1: ElastiCache automatic failover to replica (if replica survived).
          Option 2: Provision new cluster from snapshot (if snapshot < 1 hour old).
          Option 3: Provision new empty cluster and warm from origin.

T+10m  — New cluster provisioned (empty). DNS updated to point to new endpoint.

T+15m  — Circuit breaker moved to HALF-OPEN. Test requests allowed through.
          Cache miss rate: 100%. All requests fall through to origin services.
          DB load spikes — expected and acceptable for short warm-up window.
          Monitor DB CPU. If > 80%, throttle incoming traffic at load balancer.

T+20m  — Cache warming begins naturally as traffic flows through.
          Hit rate climbs: 0% → 20% → 50% → 80%.

T+30m  — Hit rate stabilizes at ~90%. DB load returns to baseline.
          Circuit breaker: CLOSED. Normal operation resumed.

T+45m  — Post-incident: trigger pre-warming job for top 1,000 products
          to accelerate hit rate recovery for next incident.
```

---

## Observability & Alerting

### Key Metrics to Track

| Metric | Source | Healthy Threshold | Alert Threshold |
|---|---|---|---|
| `cache_hit_ratio` | Application (Micrometer/Prometheus) | > 95% | < 90% for 5 min |
| `cache_evictions_total` | Redis INFO stats | Low, stable | Spike > 2x baseline |
| `redis_memory_fragmentation_ratio` | Redis INFO memory | 1.0 – 1.5 | > 1.5 (wasted memory) or < 1.0 (swap usage) |
| `cdn_cache_hit_ratio` | CDN analytics API | > 95% | < 85% for 10 min |
| `origin_latency_p99` | APM (Datadog/New Relic) | < 200ms | > 500ms for 5 min |
| `redis_connected_clients` | Redis INFO clients | Stable | Sudden drop to 0 |
| `db_connections_active` | DB metrics | < 70% of pool | > 85% of pool |

### PromQL Killer Alerts

**Alert 1: Cache hit ratio drop**
```promql
# Fires when application-level cache hit rate drops below 90%
(
  rate(cache_hits_total[5m])
  /
  (rate(cache_hits_total[5m]) + rate(cache_misses_total[5m]))
) < 0.90
```

**Alert 2: Redis eviction rate spike**
```promql
# Fires when eviction rate doubles compared to the 1-hour baseline
rate(redis_evicted_keys_total[5m])
  >
2 * rate(redis_evicted_keys_total[1h] offset 5m)
```

**Alert 3: CDN hit ratio drop (origin flood risk)**
```promql
# Fires when CDN hit ratio drops below 85% — origin is absorbing too much traffic
cdn_cache_hit_ratio{region="ap-southeast-1"} < 0.85
```

**Alert 4: Redis memory fragmentation**
```promql
# Fires when fragmentation ratio is unhealthy (too high = wasted RAM, < 1 = swap)
redis_memory_fragmentation_ratio > 1.5
  or
redis_memory_fragmentation_ratio < 1.0
```

### Per-Request Debug Log Format

Every cache interaction should emit a structured log line. This makes post-incident analysis fast.

```json
{
  "timestamp": "2024-03-15T14:23:01.234Z",
  "request_id": "req_a1b2c3d4",
  "user_id": "usr_xyz",
  "cache_layer": "redis",
  "cache_key": "cache:product:123:details",
  "cache_result": "miss",
  "ttl_remaining_ms": null,
  "fetch_source": "product_service",
  "fetch_latency_ms": 45,
  "cache_write": true,
  "region": "ap-southeast-1"
}
```

**What this enables:**
- Filter `cache_result: miss` + `region: ap-southeast-1` → instantly see if a specific region is cold.
- Filter `fetch_source: product_service` + `fetch_latency_ms > 200` → identify slow upstream services causing cache misses to be expensive.
- Aggregate `cache_result` by `cache_key` prefix → find which data types have the worst hit rates.

---

## Production Incident Analysis

### Incident: The Conversion Rate Drop

**Symptom:** At 3:00 PM on a Tuesday, the business intelligence team alerts that the checkout conversion rate has dropped from 4.2% to 1.8% over the past two hours. No deployment was made. No alerts fired.

**Diagnostic Methodology — Funnel Breakdown:**

```
Step 1: Funnel Analysis
  Homepage → Product Page → Add to Cart → Checkout → Purchase
  Conversion drop is concentrated at: Product Page → Add to Cart
  Users are landing on product pages but not adding to cart.
  Hypothesis: Product pages are slow or showing incorrect data.

Step 2: RUM (Real User Monitoring) Data
  p50 product page load time: 95ms  (normal)
  p95 product page load time: 4,200ms  ← ANOMALY
  p99 product page load time: 8,100ms  ← ANOMALY
  Slow for ~5% of users. Consistent with a partial cache failure.

Step 3: CDN Hit Ratio
  CDN hit ratio (product pages): 94%  (normal — CDN is fine)
  CDN hit ratio (product images): 96%  (normal)
  CDN is not the problem. Slow requests are reaching origin.

Step 4: Redis Metrics
  redis_hit_ratio: 70%  ← DOWN from 95% baseline
  redis_evicted_keys_total: SPIKING — 12,000 evictions/minute
  redis_memory_used_bytes: 99.8% of maxmemory
  Eviction policy: allkeys-lru

  ROOT CAUSE IDENTIFIED: Redis is full. It is evicting product cache entries
  to make room for something else. Hit rate dropped from 95% to 70%.

Step 5: What is filling Redis?
  Inspect key distribution:
  redis-cli --scan --pattern "cache:*" | awk -F: '{print $2}' | sort | uniq -c | sort -rn

  Output:
    8,420,000  wishlist
       45,000  product
        3,200  recommendations
          800  top10

  "wishlist" keys are consuming 95% of Redis memory.

Step 6: Root Cause Story
  Two days ago, the Social Wishlist feature was deployed.
  It caches each user's wishlist in Redis: cache:wishlist:{user_id}
  Average wishlist size: 2KB. 8.4 million active users.
  Total wishlist data: 8,400,000 × 2KB = ~16.8 GB
  Redis maxmemory: 16 GB

  The wishlist feature silently consumed all available Redis memory.
  LRU eviction began evicting product cache entries (smaller, less recently accessed).
  Product cache hit rate: 95% → 70%.
  30% of product page requests now hit the database.
  DB query time under load: 200ms → 4,000ms (connection pool contention).
  Users experience 4-second product page loads. They don't add to cart.
  Conversion rate drops 57%.
```

**Immediate Mitigation:**

```
1. Flush wishlist keys from Redis immediately:
   redis-cli --scan --pattern "cache:wishlist:*" | xargs redis-cli del

2. Conversion rate recovers within 5 minutes as product cache repopulates.

3. Temporarily disable wishlist caching in feature flag system.
```

**Permanent Fix:**

```
1. Namespace isolation with memory limits (Redis 7+ with key-space quotas,
   or separate Redis clusters per data domain):

   Cluster A: product-cache  (8 GB, dedicated)
   Cluster B: user-data      (8 GB: recommendations + wishlists)
   Cluster C: session-data   (4 GB)

2. Wishlist data does not belong in Redis at all for this access pattern.
   Wishlists are read infrequently (not on every product page load).
   Move to DynamoDB with DAX for caching, or PostgreSQL with a short TTL.

3. Add a pre-deploy checklist item: "Does this feature write to Redis?
   What is the estimated memory footprint at 10M users?"

4. Add alert: redis_memory_used_bytes > 80% of maxmemory → PagerDuty.
   (This incident would have been caught 6 hours earlier.)
```

**Key lesson:** Cache eviction is silent. The system does not throw an error when it evicts your data — it just stops serving it from cache. Without memory utilization alerts and key-distribution monitoring, this class of incident is invisible until it manifests as a business metric drop.

---

## Key Concepts Quick Reference

| Concept | Definition | When It Matters |
|---|---|---|
| Cache-aside (Lazy Loading) | Application checks cache first; on miss, fetches from DB and populates cache | Default pattern for most use cases |
| Write-through | Write to cache and DB simultaneously on every write | When read-after-write consistency is critical |
| Write-behind (Write-back) | Write to cache immediately; async flush to DB | High write throughput, tolerate small data loss risk |
| TTL (Time-to-Live) | Automatic expiry after a fixed duration | Universal — every cached item should have a TTL |
| Cache busting | Change the cache key (filename hash) when content changes | Static assets — enables infinite TTL |
| Cache Tags (Surrogate Keys) | Tag cached items; purge all items with a tag in one call | CDN invalidation for related content groups |
| Cache stampede | Many concurrent requests regenerate the same expired key simultaneously | High-traffic systems with popular keys |
| XFetch / Probabilistic Early Expiration | Stochastically refresh cache before expiry to avoid hard-expiration events | Eliminates stampede at the application layer |
| Request Collapsing | CDN/proxy holds N requests, sends 1 to origin, fans response to all N | Eliminates stampede at the CDN/proxy layer |
| Circuit Breaker | Fail fast when a dependency is down; prevent cascade failures | Redis failover, any critical dependency |
| LRU Eviction | Evict least-recently-used keys when memory is full | Default Redis eviction policy |
| Memory fragmentation ratio | Ratio of RSS memory to used memory; indicates allocator efficiency | Redis memory health monitoring |
| CDC (Change Data Capture) | Read DB transaction log to detect changes and trigger downstream actions | Reliable cache invalidation without application coupling |
| Fencing token / Epoch | Monotonically increasing number to reject writes from stale leaders | Split-brain prevention in distributed ownership |
| Single-home pattern | Each entity is owned by exactly one region/service; others route to it | Active-active conflict avoidance |

---

## Key Takeaways

- Caching is defense-in-depth. Each layer (browser → CDN → Redis → DB buffer pool) absorbs a specific type of load and protects the layer beneath it. Design all four layers, not just Redis.

- The hardest problem is not caching — it is invalidation. The CDC pattern (Debezium → Kafka → Invalidator) decouples invalidation from the application and makes it reliable. Any committed DB change will eventually trigger an invalidation, even if the application crashes mid-operation.

- Cache stampede is a CDN/proxy problem first, an application problem second. Fix it at the CDN layer with request collapsing. Use XFetch at the application layer as a secondary defense. Never rely on SETNX mutex under high concurrency — it trades DB overload for application server starvation.

- Redis failover math is unforgiving. A 95% hit rate means a Redis outage causes a 10x DB overload. "Fail open to DB" is not a strategy — it is a guaranteed outage. Use read replicas as the primary defense and circuit breakers as the final backstop.

- Cache key ownership must be explicit. Two services writing to the same key is a race condition waiting to happen. Split keys by ownership boundary and compose in the application layer using pipelining.

- Memory is the silent killer. Redis eviction does not throw errors — it silently drops your data. Without memory utilization alerts and key-distribution monitoring, a new feature can consume all cache memory and cause a business-metric incident with no technical alerts firing.

- Multi-region caching is a spectrum. Active invalidation (SNS/SQS fan-out) meets tight SLAs but adds operational complexity. TTL-only is simple and sufficient for most data. Immutable versioned keys sidestep invalidation entirely for static content. Use the right tool for each data type's staleness tolerance.

