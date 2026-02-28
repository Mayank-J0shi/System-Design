# Rate Limiting & Abuse Mitigation

A deep-dive into designing a production-grade rate limiting and abuse mitigation layer for a high-traffic fintech API platform — covering algorithms, data layer design, distributed systems tradeoffs, failure modes, and operational concerns.

---

## Table of Contents

1. [Problem Context](#problem-context)
2. [Requirements & Constraints](#requirements--constraints)
3. [Where the Logic Lives](#where-the-logic-lives)
4. [Client Identification Strategy](#client-identification-strategy)
5. [Rate Limiting Algorithms](#rate-limiting-algorithms)
6. [Data Layer: Redis & Atomicity](#data-layer-redis--atomicity)
7. [Failure Modes: Fail-Open vs Fail-Closed](#failure-modes-fail-open-vs-fail-closed)
8. [Multi-Region Deployments](#multi-region-deployments)
9. [Dynamic Configuration](#dynamic-configuration)
10. [Abuse Mitigation: Beyond Rate Limiting](#abuse-mitigation-beyond-rate-limiting)
11. [Observability & Alerting](#observability--alerting)
12. [Operational Runbooks & Post-Mortem Learnings](#operational-runbooks--post-mortem-learnings)

---

## Problem Context

A fintech platform exposes a public-facing REST API consumed by:
- **Mobile apps** — consumer login, OTP verification, checkout
- **Third-party partners** — payment initiation, account queries (contractual SLAs)
- **Internal microservices** — fraud checks, notification triggers

Real-world failure scenarios this system must handle:
- A credential stuffing attack: 50k req/min to `/auth/login` from ~10,000 distinct IPs
- A rogue partner integration loop: thousands of duplicate `/payments/initiate` calls from a single API key
- Legitimate users throttled during a flash sale on `/checkout` due to a misconfigured global limit

---

## Requirements & Constraints

### Functional
- Per-user, per-partner, and per-endpoint rate limits
- Tiered SLAs for partners: Bronze (100 rps), Silver (500 rps), Gold (2000 rps)
- Dynamic limit configuration — adjustable at runtime without redeployment
- Temporary overrides for planned high-traffic events (flash sales)
- Machine-readable `429` responses with `Retry-After` header for partners
- Human-readable error messages for mobile users

### Non-Functional
- Latency budget for the rate limiting check: **< 5ms p99**
- Scale: ~500k req/min across the platform
- GDPR compliance — IP addresses are PII, retention must be limited
- High availability — rate limiting failures must not take down the platform

---

## Where the Logic Lives

### Decision: API Gateway (Kong)

The rate limiting logic lives inside the **API Gateway** as a plugin, not as a separate microservice.

```
Client Request
      │
      ▼
┌─────────────────────────────────────────────────────┐
│                  Kong API Gateway                   │
│                                                     │
│  1. Extract identifier (JWT claim / API key / cert) │
│  2. Execute rate limit plugin                       │
│     └── Atomic check+increment against Redis        │
│  3a. Allow → proxy to upstream service              │
│  3b. Deny  → return 429 immediately (no upstream)   │
└─────────────────────────────────────────────────────┘
      │
      ▼
  Upstream Service
  (never sees throttled requests)
```

**Why the gateway, not a separate rate-limiter service?**

A dedicated rate-limiter microservice would require:
```
Client → Kong → Rate Limiter Service → Kong → Upstream
```

This adds a full synchronous network hop on every single request. Even with gRPC and co-located services in the same AZ, consistently achieving sub-5ms p99 becomes fragile. It also doubles the points of network failure. Given the strict latency requirement, this is not viable.

**Tradeoffs of the gateway approach:**

| Pro | Con |
|-----|-----|
| Minimal latency overhead (Kong + Redis in same VPC = 1-2ms) | Custom logic requires Lua plugin development |
| Centralized enforcement — no inconsistency across services | Gateway becomes a critical path dependency |
| Downstream services stay clean, no rate-limit code | Lua is a specialized skillset |
| Single place to audit, debug, and change policy | |

---

## Client Identification Strategy

The choice of identifier determines fairness and precision. A single strategy does not work for all client types.

### Mobile Users (Authenticated)

**Identifier: `user_id` from the JWT claim**

Kong decodes the JWT and extracts the `sub` or `user_id` claim without calling an external service. This ties usage to a specific authenticated user, not a network address.

Why not IP? Many users share IPs — corporate NATs, university networks, mobile carrier NAT. Rate limiting by IP punishes an entire network for one user's behavior.

### Mobile Users (Unauthenticated — e.g., `/auth/login`)

**Identifier: Source IP Address + Global Endpoint Limit**

No JWT is present before authentication, so IP is the only available signal. This is inherently imprecise but unavoidable.

To compensate, a **global endpoint-level circuit breaker** is added on top of per-IP limits:

```
Per-IP limit:       5 req/min per IP
Global limit:    2000 req/min total on /auth/login
```

The global limit acts as a circuit breaker. Even if a distributed attack (10,000 IPs × 5 req/min = 50,000 req/min) bypasses per-IP limits, the global limit caps total load on the auth service. The tradeoff is that legitimate users may be collaterally throttled during an active attack — this is an acceptable degradation compared to the auth service going down entirely.

### Third-Party Partners

**Identifier: API Key from request header**

The API key uniquely identifies the partner and maps to their contracted SLA tier. Kong reads the `X-API-Key` header, uses it as the Redis key, and applies the tier-specific limit (100/500/2000 rps).

### Internal Microservices

**Identifier: Service Identity from mTLS certificate**

In Kubernetes, pod IPs are ephemeral and meaningless as identifiers. mTLS provides a cryptographically secure, stable service identity (e.g., `fraud-check-service.prod.svc.cluster.local`).

The intent for internal services is not to throttle them but to provide a **safety net**. Limits are set well above normal operating peaks. This prevents a bug (e.g., an infinite retry loop in one service) from causing a cascading failure by overwhelming another service.

---

## Rate Limiting Algorithms

### Fixed Window Counter

A counter per fixed time window (e.g., per minute). Resets at the start of each window.

```
Window:  |--- 10:00 ---|--- 10:01 ---|
Count:       1800           200
```

**Problem — Edge Burst:**
A client can send 2000 requests at `10:00:59` and another 2000 at `10:01:01`. That's 4000 requests in 2 seconds — double the allowed rate — and neither window's counter exceeds the limit.

**Memory:** Very low (one integer per key per window).
**Accuracy:** Poor. Dangerous for protecting downstream services.

---

### Sliding Window Log

For every request, store its timestamp in a sorted set. To check the limit, count timestamps within the last N seconds.

**Accuracy:** Perfect. Every request is individually tracked.

**Problem — Memory:**
To enforce 2000 rps for one Gold partner, you store 2000 timestamps per second. At 500k req/min platform-wide, this is ~8,333 timestamps/second across all users. Redis memory requirements become impractical at scale.

**Memory:** Extremely high. Not operationally viable at this scale.

---

### Sliding Window Counter (Chosen)

A hybrid approach. Track two counters: current window and previous window. Estimate the rate using a weighted combination.

```
Window size: 60 seconds
Current position: 15 seconds into the current minute

Previous window count: 1800
Current window count:  400

Overlap of previous window still in the 60s view:
  (60 - 15) / 60 = 75%

Estimated rate = (1800 × 0.75) + 400 = 1750
```

**Accuracy:** Very good. Smooths the edge burst problem. The maximum burst is close to the actual limit, not 2x.

**Memory:** Very low. Two integers per identifier (stored as a Redis Hash).

**Why this is the right choice:**

| Algorithm | Memory | Accuracy | Viable at Scale |
|-----------|--------|----------|-----------------|
| Fixed Window | Low | Poor (2x burst) | Yes, but risky |
| Sliding Window Log | Very High | Perfect | No |
| Sliding Window Counter | Low | Good | Yes ✅ |

The sliding window log is academically perfect but operationally unviable. The fixed window is too risky for a fintech platform where a 2x burst can take down a downstream service. The sliding window counter is the pragmatic, production-grade choice.

---

## Data Layer: Redis & Atomicity

### The Race Condition Problem

With 10 Kong nodes independently processing requests for the same Gold partner (2000 rps limit), a naive `GET → CHECK → INCR` pattern fails:

```
Current count: 1999

T1:     Node A runs GET → sees 1999
T1+0.1: Node B runs GET → sees 1999 (Node A hasn't incremented yet)
T1+0.5: Node A sees 1999 < 2000 → allows request → runs INCR → count = 2000
T1+0.6: Node B sees 1999 < 2000 → allows request → runs INCR → count = 2001
```

Both nodes admitted the request. With 10 nodes at the boundary, over-admission can be significant — directly violating partner SLAs and potentially overwhelming downstream services.

### Solution: Atomic Lua Script

Redis guarantees that a Lua script executes atomically. No other Redis command can run while the script is executing. This eliminates the race condition entirely.

```lua
-- KEYS[1] = rate limit key (e.g., "ratelimit:api_key_abc:1678886400")
-- ARGV[1] = limit (e.g., 2000)
-- ARGV[2] = window expiry in seconds (e.g., 1)

local current_count = redis.call('INCR', KEYS[1])

-- Set expiry only on first increment (avoids resetting TTL on every call)
if current_count == 1 then
    redis.call('EXPIRE', KEYS[1], ARGV[2])
end

if current_count > tonumber(ARGV[1]) then
    return 0  -- Deny
else
    return 1  -- Allow
end
```

**Note:** This script increments before checking. The token is consumed even on a denied request. This is a conscious tradeoff — it simplifies the script and is acceptable in most production systems, but worth knowing explicitly.

**Why Lua over a Redis transaction (MULTI/EXEC)?**

A transaction with `WATCH` still requires multiple round trips (WATCH → GET → MULTI → INCR → EXEC). The Lua script is a single `EVAL` call — one network round trip, guaranteed atomic execution. This is both simpler and faster.

### Sliding Window Counter in Redis

For the sliding window counter, the Kong plugin manages two keys per identifier:

```
ratelimit:{identifier}:prev  → count in previous window
ratelimit:{identifier}:curr  → count in current window
```

The Lua script:
1. Determines the current window timestamp
2. Checks if the current key has rolled over to a new window
3. If so, copies `curr` to `prev` and resets `curr`
4. Increments `curr`
5. Calculates the weighted estimate
6. Returns allow/deny

### Redis Cluster Sizing

At 500k req/min, each request triggers one `EVAL` call to Redis. That's ~8,333 ops/sec. A single Redis node handles hundreds of thousands of ops/sec, so a small cluster (3-5 nodes with replication) is sufficient. The key concern is latency, not throughput — co-locate Redis in the same AZ as Kong nodes.

---

## Failure Modes: Fail-Open vs Fail-Closed

A blanket fail-open or fail-closed policy is naive. The correct approach is a **per-endpoint failure policy** based on the risk profile of the protected service.

### High-Risk Endpoints: Fail-Closed with Local Fallback

**Endpoints:** `/auth/login`, `/payments/initiate`, any endpoint that mutates critical data or has high abuse potential.

**Risk profile:** The cost of a false negative (letting a malicious request through) is catastrophic — account takeovers, financial fraud, cascading failures. This is far worse than the cost of a false positive (blocking a legitimate user during an outage).

**Implementation:**

When the Kong plugin cannot reach Redis within its timeout, it does **not** simply block all traffic (that's a full outage). Instead, it falls back to a **local, in-memory, per-node rate limiter**:

```
Redis unavailable
        │
        ▼
Local in-memory counter (per Kong node)
  └── Apply conservative emergency limits
      e.g., 10 req/5min per IP on /auth/login
        │
        ▼
Log critical error + alert on-call
```

**What this achieves:**
- Stops the rogue partner loop immediately (single-source traffic)
- Significantly mitigates distributed attacks (limits each of 10,000 IPs)
- Buys the on-call team time to restore Redis
- Turns a potential catastrophe into a manageable degradation

**Known limitation:** The local fallback is per-node and not globally coordinated. A distributed attack at exactly the per-node limit threshold could still slip through. It is a mitigation, not a complete solution.

### Low-Risk Endpoints: Fail-Open

**Endpoints:** Read-only endpoints like `/products/search`, `/account/balance`, static content.

**Risk profile:** The cost of a false positive (blocking a legitimate user) is higher than a false negative. Blocking all users from browsing products during a Redis outage is worse than allowing excess search queries for a few minutes. Read-only services are generally more resilient to load spikes.

**Implementation:** Log a critical error, allow the request to pass.

### Configuration

The failure mode is stored per-route in the configuration store:

```json
{
  "route": "/auth/login",
  "limit": 5,
  "window": "60s",
  "fail_mode": "closed"
}

{
  "route": "/products/search",
  "limit": 100,
  "window": "60s",
  "fail_mode": "open"
}
```

---

## Multi-Region Deployments

### The Problem: Global vs Per-Region Limits

To enforce a true global 2000 rps limit for a Gold partner across `us-east-1` and `eu-west-1`, every Kong node in both regions would need to synchronously query a single centralized Redis instance. Cross-continent network latency is 70-100ms. This violates the 5ms latency budget by an order of magnitude. A globally consistent rate limiter is architecturally incompatible with a sub-5ms latency requirement.

### Decision: Per-Region Limits with Async Global Monitoring

Each region has its own independent Kong cluster and co-located Redis cluster. A Gold partner's API key has a 2000 rps limit in `us-east-1` AND a 2000 rps limit in `eu-west-1`.

```
us-east-1                          eu-west-1
┌──────────────────────┐          ┌──────────────────────┐
│  Kong Cluster        │          │  Kong Cluster        │
│  Redis (local)       │          │  Redis (local)       │
│  Limit: 2000 rps     │          │  Limit: 2000 rps     │
└──────────────────────┘          └──────────────────────┘
         │                                  │
         └──────────────┬───────────────────┘
                        │ (async, out-of-band)
                        ▼
              Central Data Warehouse
              (BigQuery / Snowflake)
              Global usage analytics
              Business intelligence alerts
```

**Tradeoffs:**

| Concern | Reality |
|---------|---------|
| Partner can achieve 4000 rps globally | Addressed contractually: SLA guarantees "2000 rps per region" |
| No real-time global enforcement | Async monitoring detects abuse patterns within 5-10 minutes |
| Operational simplicity | Each region is fully independent — an EU Redis outage has zero impact on US traffic |

**Async global monitoring:** All Kong access logs (API key, endpoint, allow/deny decision) stream to a central data warehouse. Analytics queries run every 5-10 minutes to calculate true global usage per partner. A partner consistently using 3900 rps globally triggers a business intelligence alert for the account management team — not a real-time block, but a signal to investigate or upsell to a higher global tier.

This separates real-time enforcement (latency-sensitive, per-region) from business-level monitoring (latency-tolerant, global).

---

## Dynamic Configuration

### Requirements

- On-call engineers must be able to adjust limits in real-time without a deployment
- Marketing/Operations must be able to schedule temporary overrides for planned events
- All changes must be auditable
- Temporary overrides must automatically revert — no manual cleanup required

### Architecture: etcd with Watch-Based Push

**Source of truth: `etcd`**

A highly-available key-value store holds all rate limiting configuration:

```
config/ratelimits/partners/api_key_xyz
  → {"limit": 2000, "window": "1s", "fail_mode": "closed"}

config/ratelimits/endpoints/auth_login
  → {"limit": 2000, "window": "60s", "fail_mode": "closed"}

config/ratelimits/overrides/flash_sale_checkout
  → {"limit": 10000, "window": "60s", "expiry": "2024-12-25T18:00:00Z"}
```

**Propagation: Push, not Poll**

Each Kong node establishes a persistent watch on the `/config/ratelimits/` prefix at startup. When a value changes in etcd, every connected Kong node receives the update immediately and loads the new rules into local memory.

```
Engineer updates etcd
        │
        ▼ (milliseconds)
etcd pushes change notification to all watchers
        │
        ├──▶ Kong Node 1 updates in-memory config
        ├──▶ Kong Node 2 updates in-memory config
        └──▶ Kong Node N updates in-memory config
```

**Consistency model:** Eventual consistency, measured in milliseconds. For a brief moment, one node may use the old limit while others have the new one. This is acceptable — the system converges almost instantaneously, and the risk of momentary inconsistency is negligible compared to the operational benefit of near-instant propagation.

### Scheduled Overrides API

Direct etcd edits are error-prone for planned events. A dedicated internal service wraps etcd with:

- **Mandatory `expiry` field** — overrides cannot be created without an end time
- **Automatic reversion** — a background job monitors expiry timestamps and removes overrides when they expire
- **Audit trail** — all changes logged with who made them and when
- **Self-service** — authorized non-engineering teams (Marketing, Operations) can schedule overrides without involving on-call engineers

```
POST /overrides
{
  "endpoint": "/checkout",
  "limit": 10000,
  "window": "60s",
  "start_time": "2024-12-25T10:00:00Z",
  "end_time":   "2024-12-25T18:00:00Z",
  "reason":     "Christmas flash sale",
  "requested_by": "marketing-team"
}
```

This transforms the process from "remember to do X and remember to undo X" to "schedule X with a guaranteed expiry."

---

## Abuse Mitigation: Beyond Rate Limiting

Simple rate limiting has a ceiling. A credential stuffing attack from 10,000 distinct IPs, each sending 5 req/min (well within per-IP limits), saturates the global endpoint limit and causes collateral damage to legitimate users. A different tool is needed.

### From Rate-Based to Risk-Based Blocking

For every request to a sensitive endpoint, calculate a **risk score** from multiple signals. Act on the score rather than just the count.

```
Risk Score → Action

0–30:   Allow (looks legitimate)
31–70:  Challenge (suspicious — increase cost of interaction)
71–100: Block (highly likely malicious — 403 + penalty box)
```

### Signal 1: Client Integrity (User-Agent)

Our legitimate mobile apps use a specific, known User-Agent string (e.g., `MyFintechApp/3.4.1 (iOS 16.5; iPhone14,5)`). Attack scripts typically use generic agents or none at all.

```
User-Agent matches known app pattern:  Score +0
User-Agent is a known script/bot:      Score +40
User-Agent is missing or generic:      Score +20
```

**Caveat:** User-Agent spoofing is trivial. This is a useful signal but low-confidence on its own. Weight it lower than behavioral signals.

### Signal 2: IP Reputation (ASN Lookup)

Legitimate users come from residential or mobile carrier IPs. Attackers commonly use data centers, VPNs, or TOR exit nodes. A local MaxMind GeoIP2/ASN database lookup is done in-memory — no network call required, sub-millisecond latency.

```
IP from known data center (AWS, GCP, Azure, DigitalOcean):  Score +30
IP is a known TOR exit node or public proxy:                 Score +50
IP from residential/mobile ISP:                             Score +0
```

### Signal 3: Behavioral History (Failure Feedback Loop)

When the `/auth` service sees a failed login (`Invalid Credentials`), it asynchronously reports the failure event (via Kafka or fire-and-forget UDP) along with the source IP to a reputation service. This service maintains failure counts per IP in Redis.

```
HINCRBY ip_reputation:{ip} failed_logins 1
EXPIRE  ip_reputation:{ip} 3600  -- 1 hour window
```

The Kong plugin makes a pipelined (non-sequential) Redis call to fetch this reputation alongside the rate limit check.

```
IP has >10 failed logins in the last hour:   Score +20
IP has >100 failed logins in the last hour:  Score +40
```

**Implementation note:** This second Redis call must be pipelined with the rate limit check, not sequenced after it. Both calls are issued simultaneously; the plugin waits for both responses together. This keeps the total Redis overhead within the latency budget.

### The Challenge Mechanism (Medium Risk: Score 31–70)

Blocking a request with a medium risk score causes collateral damage to legitimate users. Instead, **increase the cost of the interaction** without blocking it outright.

**For Mobile Apps — Device Attestation:**

The API responds with `401 Unauthorized` and a `challenge-required` body. The mobile app triggers a native device attestation check (Apple App Attest / Android Play Integrity). The app re-submits the login request with the attestation token. The gateway validates the token.

A script or bot cannot pass device attestation. A real device on a real OS can, transparently to the user. This creates an express lane for legitimate clients while making the attack significantly more expensive.

**Cold-start problem:** New installs haven't established reputation yet. For first-time requests with no history, treat the device attestation result as the primary signal. A new install that passes attestation gets a clean slate.

**For Partners/Web:**

A medium risk score for a partner API triggers a security team alert but still allows the request (we have a contractual relationship and can communicate directly). For web flows, this is where a CAPTCHA challenge is appropriate.

### Architecture: Extended Kong Plugin

All of this logic lives inside the same Kong plugin — not a separate service. The flow on every request to a sensitive endpoint:

```
Request arrives at Kong
        │
        ▼
Step 1: Basic rate limit check (existing)
  └── Per-IP or per-user sliding window counter
  └── If exceeded → 429 immediately
        │
        ▼ (if basic limit passes)
Step 2: Risk scoring (new)
  ├── Check User-Agent (0ms, in-memory)
  ├── ASN lookup in MaxMind DB (<1ms, in-memory)
  └── Fetch IP failure history from Redis (1-2ms, pipelined with step 1)
        │
        ▼
Step 3: Score evaluation
  ├── 0–30:   Allow
  ├── 31–70:  Return challenge response
  └── 71–100: Block (403) + add IP to penalty box in Redis (TTL: 1 hour)
```

Total overhead: comfortably within the 5ms budget.

### Why This Works

- Raises the bar for attackers: rotating IPs is no longer sufficient. They now need clean residential IPs, spoofed User-Agents, and the ability to bypass device attestation — a much harder and more expensive problem.
- Protects legitimate users: the challenge step creates an express lane for real devices. The global limit is no longer saturated by low-quality attack traffic.
- Incremental design: no ML platform required. High-signal, low-latency deterministic checks added to an existing enforcement layer. The behavioral feedback loop is the first step toward a self-tuning system.

---

## Observability & Alerting

### The Critical Metric: Dimensional Throttle Reason

A generic `requests_throttled_total` counter is insufficient. It cannot distinguish between "good" throttling (blocking an attacker) and "bad" throttling (blocking legitimate users due to misconfiguration).

The essential metric is:

```
rate_limited_requests_total{endpoint, client_id, reason}
```

Where `reason` is populated by the Kong plugin with the specific limit that triggered the block:

```
reason="per_user_limit"
reason="per_ip_limit"
reason="global_endpoint_limit"
reason="risk_score_block"
reason="penalty_box"
```

### Killer Alerts

**Alert: Global limit firing while per-user limits are not**

This is the unique, unambiguous signature of a misconfigured global limit blocking legitimate users (as opposed to an attack being correctly blocked).

```promql
# Alert fires when global endpoint limit is the cause of throttling
# but individual user limits are not being hit
sum(rate(rate_limited_requests_total{
  endpoint="/checkout",
  reason="global_endpoint_limit"
}[1m])) > 100

AND

sum(rate(rate_limited_requests_total{
  endpoint="/checkout",
  reason="per_user_limit"
}[1m])) < 10
```

An on-call engineer seeing this alert knows immediately that the global limit configuration is the problem — not the checkout service, not a user-side issue. Time to resolution drops from 20 minutes to 60 seconds.

**Alert: Distributed attack pattern**

```promql
# High volume of risk_score_block from many distinct IPs
# signals a distributed attack, not a single bad actor
count(rate_limited_requests_total{
  endpoint="/auth/login",
  reason="risk_score_block"
} by (client_id)) > 500
```

**Alert: Redis unavailability causing fallback activation**

```promql
# Kong plugin falling back to local in-memory limiter
sum(rate(rate_limit_redis_errors_total[1m])) > 10
```

### Debugging Tooling

When a partner reports being incorrectly blocked, the investigation requires structured logs that capture, for each request:

```json
{
  "timestamp": "2024-12-27T10:45:00Z",
  "endpoint": "/payments/initiate",
  "client_id": "api_key_xyz",
  "client_type": "partner",
  "tier": "gold",
  "limit_applied": 2000,
  "current_count": 2001,
  "decision": "deny",
  "reason": "per_partner_limit",
  "risk_score": 0,
  "region": "us-east-1",
  "kong_node": "kong-pod-7d9f8b"
}
```

This log entry answers every question in a single record: what identifier was used, what limit was applied, what the count was, and why the request was denied.

---

## Operational Runbooks & Post-Mortem Learnings

### Post-Mortem: Flash Sale Throttling Incident

**What happened:** During a peak flash sale, legitimate customers received 429s on `/checkout` for 20 minutes. The checkout service was healthy (CPU at 15%). The global endpoint limit had not been raised for the event.

**Root cause analysis (5 Whys):**

1. Why were customers blocked? → The global `/checkout` limit was hit by legitimate surge traffic.
2. Why was the global limit hit? → Flash sale traffic exceeded the peacetime global limit, even though no individual user exceeded their personal limit.
3. Why wasn't the limit raised? → A manual etcd update was missed.
4. Why did we rely on a manual process for a revenue-critical event? → **Systemic failure: no formal, verifiable pre-event checklist with ownership.**
5. Why didn't alerting catch this immediately? → **Systemic failure: monitoring lacked the `reason` dimension to distinguish misconfiguration from attack.**

**The failure was not human error. It was a system that made human error easy and made detection slow.**

### Process Changes

**Flash Sale Readiness Checklist:**

A mandatory, version-controlled checklist for all planned high-traffic events. Must be reviewed and signed off by a designated "Event Commander" before the event begins. Line items include:

- [ ] Rate limit overrides applied and verified in staging
- [ ] Override expiry time set and confirmed
- [ ] On-call engineer designated for event window
- [ ] Rollback procedure documented
- [ ] Monitoring dashboards bookmarked and reviewed

**Pre-Mortems:**

For all major events, hold a pre-mortem: "How could this fail?" Run dry runs of the readiness checklist in staging to build muscle memory.

### Technical Changes

**Scheduled Overrides API** (described in [Dynamic Configuration](#dynamic-configuration)):

Replaces direct etcd edits with a service that enforces mandatory expiry and automatic reversion. Transforms "remember to do X and undo X" into "schedule X with guaranteed cleanup."

**Adaptive Rate Limiting (Future):**

When the global limit on an endpoint approaches capacity, analyze the quality of incoming traffic. If >98% of requests have valid JWTs, clean residential IPs, and zero risk flags, the system can autonomously scale the global limit by a small margin (e.g., +10%) as a pressure-release valve.

**Important caveat:** Autonomous limit scaling in a fintech context is a hard sell to a security team. Any self-adjustment must have a tight ceiling, a human approval step or kill switch, and comprehensive audit logging. The tradeoff between operational convenience and security posture must be explicitly accepted by the security team before implementation.
