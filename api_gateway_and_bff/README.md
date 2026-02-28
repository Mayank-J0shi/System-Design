# API Gateway & BFF — Interview Learning Notes

> Senior Engineer level. Covers design, operations, failure modes, and tradeoffs.
[ref]https://chat.swiggy.cloud/s/165dd015-6b94-41ca-9853-2792f43215cc

---

## Table of Contents

1. [Problem Statement & Mental Model](#1-problem-statement--mental-model)
2. [API Gateway — Responsibilities](#2-api-gateway--responsibilities)
3. [BFF Pattern — Responsibilities & Boundaries](#3-bff-pattern--responsibilities--boundaries)
4. [Drawing the Line: Gateway vs BFF](#4-drawing-the-line-gateway-vs-bff)
5. [BFF Ownership Models](#5-bff-ownership-models)
6. [Authentication & Authorization at Scale](#6-authentication--authorization-at-scale)
7. [High Availability & Deployment Topology](#7-high-availability--deployment-topology)
8. [Distributed Rate Limiting](#8-distributed-rate-limiting)
9. [Zero-Downtime Config Changes](#9-zero-downtime-config-changes)
10. [Partner API BFF — Special Concerns](#10-partner-api-bff--special-concerns)
11. [API Versioning & Deprecation Lifecycle](#11-api-versioning--deprecation-lifecycle)
12. [Retry Storm Protection](#12-retry-storm-protection)
13. [Data Plane vs Control Plane](#13-data-plane-vs-control-plane)
14. [Resilience Config Placement](#14-resilience-config-placement)
15. [Production Incident Playbook](#15-production-incident-playbook)
16. [Architectural Tradeoffs Summary](#16-architectural-tradeoffs-summary)
17. [Gaps to Sharpen (Staff-Level Delta)](#17-gaps-to-sharpen-staff-level-delta)

---

## 1. Problem Statement & Mental Model

### Symptoms that signal you need a Gateway + BFF layer

- Clients making 6-8 API calls per screen ("chatty client" problem)
- No consistent place to enforce auth — each service does it differently or not at all
- Rate limiting inconsistent across 30 services
- No single entry point — hard to onboard new developers or partners
- Cross-cutting concerns (logging, tracing, metrics) scattered and inconsistent

### Core Problems Being Solved

| Problem | Root Cause | Solution |
|---|---|---|
| Poor mobile performance | Multiple round trips per screen | BFF aggregation |
| Security inconsistency | Distributed auth enforcement | Gateway as single security checkpoint |
| Operational chaos | 30 teams implementing cross-cutting concerns differently | Centralized gateway policies |
| No coherent API surface | Direct client-to-service calls | Gateway as "front door" |


---

## 2. API Gateway — Responsibilities

The gateway is the **single, authoritative entry point** for all inbound traffic. It handles **generic, cross-cutting concerns that apply to every request regardless of client**.

### Core Responsibilities

**1. Request Routing**
- Inspects path, headers, method and forwards to the correct downstream service or BFF
- The "switchboard" function — dumb routing, no business logic

**2. Authentication (Centralized Security)**
- Validates JWTs, API keys, OAuth tokens
- Performs coarse-grained authorization (e.g., "Is this a paying customer?")
- Rejects unauthenticated requests immediately — they never touch internal services
- Does NOT perform fine-grained authorization (e.g., "Does user A own order 123?") — that belongs to the owning service

**3. Global Rate Limiting & Throttling**
- System-wide rules to protect the entire backend
- e.g., "No single IP > 100 req/sec"
- Per-partner quotas for external APIs (a business feature, not just safety)

**4. SSL/TLS Termination**
- Handles cryptographic overhead at the edge
- Internal services communicate over HTTP within the trusted VPC
- Reduces CPU load on every downstream service

**5. Observability Injection**
- Logs every incoming request and outcome in one consistent place
- Initiates a trace ID (e.g., `X-Trace-ID`) passed downstream to all microservice calls
- Single source of truth for request-level metrics (RPS, error rates, latency)

**6. Protocol Translation (optional)**
- Can translate between external protocols (REST, GraphQL) and internal protocols (gRPC)
- More commonly done at the BFF layer for client-specific needs


---

## 3. BFF Pattern — Responsibilities & Boundaries

BFF = Backend for Frontend. One BFF per client type:
- Mobile BFF
- Web BFF
- Partner/Third-party BFF

The BFF's job is **client-specific aggregation and transformation**. It is NOT a generic service.

### Core Responsibilities

**1. Aggregation & Orchestration**
- Receives one request from the client (e.g., `GET /v1/home-screen`)
- Makes 6-8 parallel or sequential calls to downstream microservices
- Waits for all responses, combines them, returns a single payload
- Directly solves the "chatty client" problem

**2. Data Transformation & Shaping**
- Trims large backend objects to only the fields the UI needs
- Combines fields (e.g., `firstName + lastName → fullName`)
- Reduces payload size, simplifies client-side logic
- Different clients need different shapes of the same data

**3. Protocol Translation**
- Web BFF might expose GraphQL while internal services are REST
- Mobile BFF might use REST while internal services use gRPC
- Translation lives here, not in the downstream services

**4. Error Mapping**
- Translates a 500 from InventoryService + 404 from PromoService into a single, user-friendly error message
- Clients should not see raw internal error codes

### What Does NOT Belong in the BFF

This is the most common failure mode. The BFF becomes a "mini-monolith" when it accumulates business logic.

**Red flag:** Complex, stateful, reusable orchestration (e.g., checkout flow: validate cart → reserve inventory → apply promo → authorize payment → create order).

**Rule:** If the same orchestration logic would need to be duplicated across two BFFs, it does not belong in either. Extract it to a dedicated **Workflow/Orchestrator Microservice**. The BFF then makes one call to `CheckoutOrchestratorService`.

**The BFF's job is to orchestrate data for a view, not to orchestrate business processes.**


---

## 4. Drawing the Line: Gateway vs BFF

```
Mobile App → API Gateway → Mobile BFF → Downstream Microservices
Web App    → API Gateway → Web BFF    → Downstream Microservices
Partner    → API Gateway → Partner BFF → Downstream Microservices
```

**The demarcation question: "Is this concern generic or client-specific?"**

| Concern | Lives In | Reason |
|---|---|---|
| Auth token validation | Gateway | Applies to every request from every client |
| Global rate limiting | Gateway | Protects the entire backend |
| TLS termination | Gateway | Infrastructure concern, not client-specific |
| Request tracing initiation | Gateway | Cross-cutting, applies everywhere |
| Home screen aggregation | Mobile BFF | Specific to mobile UI layout |
| GraphQL schema | Web BFF | Specific to web client's data needs |
| Payload trimming for mobile | Mobile BFF | Mobile needs smaller payloads than web |
| Partner quota enforcement | Gateway + Partner BFF | Gateway for hard limits, BFF for partner-specific logic |
| Fine-grained authorization | Owning microservice | Only the data owner knows the rules |

---

## 5. BFF Ownership Models

| Owner | Pros | Cons |
|---|---|---|
| Frontend Team | High autonomy, fast iteration, closest to client needs | Potential skill gap in backend resilience/observability |
| Central Backend Team | High quality, consistent implementation | Bottleneck — every field change requires a ticket and handoff |
| Shared Platform Team | Enables other teams, owns tooling not features | Can become "ivory tower" if disconnected from feature teams |

### Recommended: The "Paved Road" Hybrid Model

- **Frontend team owns the BFF's business logic** — they write aggregation and transformation code
- **Platform team owns the "paved road"** — they provide:
  - BFF service template (pre-configured Spring Boot / Express.js project)
  - Standardized libraries for service discovery, resilient HTTP (retries, circuit breakers), logging, metrics, tracing
  - CI/CD pipelines with automated testing, security scanning, deployment
  - Clear documentation and guardrails

Frontend teams get autonomy. Platform team ensures consistency and quality. Neither is a bottleneck to the other.


---

## 6. Authentication & Authorization at Scale

### The Two-Layer Model

**Layer 1 — Gateway: Authentication + Coarse-grained Authorization**
- Validates JWT signature and expiry
- Confirms the user is who they say they are
- Checks broad permissions (e.g., "Is this user active? Is this a valid API key?")
- Extracts user identity and injects it as a header (e.g., `X-User-Context: {userId, roles}`) for downstream services

**Layer 2 — Owning Microservice: Fine-grained Authorization**
- `GET /orders/{orderId}` — only the Order Service knows if user A owns order 123
- The service that owns the data is the ultimate authority on who can access it
- The BFF must NOT perform this check — it would break encapsulation and create coupling

### The Scale Problem: 30 Services, Each Writing Their Own Auth Logic

If every service has hardcoded `if/else` authorization rules:
- Duplicated logic across 30 codebases
- Inconsistent behavior (Java service vs Go service may interpret rules differently)
- No central audit trail
- Changing a policy requires redeploying multiple services

### Solution: Centralized Policy Engine (OPA — Open Policy Agent)

**Principle: Decouple the policy decision from the policy enforcement.**

```
Request Flow:
  Gateway → validates JWT → injects X-User-Context header
  BFF     → passes X-User-Context through on all downstream calls
  Service → fetches resource from DB
          → queries OPA: "Can user {id, roles} perform {action} on {resource}?"
          → OPA returns: {"allow": true/false}
          → Service enforces the decision (403 if denied)
```

**OPA Architecture:**
- Runs as a sidecar alongside each microservice (lightweight, local network call)
- Policies written in **Rego** (declarative language)
- e.g., `allow if user.id == resource.owner_id` or `allow if "admin" in user.roles`
- Policies stored centrally, pushed to all OPA sidecars — no service redeployment needed

**Why OPA solves the scale problem:**

| Problem | OPA Solution |
|---|---|
| Duplicated logic | Policies live in one place, evaluated by one engine |
| Language inconsistency | OPA is just an API — language agnostic |
| No audit trail | All policy evaluations logged centrally |
| Policy changes require redeploy | Update Rego policies, push to OPA — services unchanged |
| Encapsulation preserved | Service still fetches its own data to provide as context to OPA |


---

## 7. High Availability & Deployment Topology

The gateway is a **Tier-0 service**. Its failure is a total platform outage. Design for redundancy at every layer.

### Multi-Layer Redundancy

```
                    ┌─────────────────────────────┐
                    │   Global Load Balancer       │
                    │ (AWS Global Accelerator /    │
                    │  Cloudflare)                 │
                    │ Health-checks regions,       │
                    │ routes to lowest-latency     │
                    └──────────┬──────────────────┘
                               │
              ┌────────────────┴────────────────┐
              │                                 │
     ┌────────▼────────┐               ┌────────▼────────┐
     │   us-east-1     │               │   us-west-2     │
     │  Regional ALB   │               │  Regional ALB   │
     └────────┬────────┘               └────────┬────────┘
              │                                 │
    ┌─────────┼─────────┐             ┌─────────┼─────────┐
    │         │         │             │         │         │
  AZ-1a     AZ-1b     AZ-1c        AZ-2a     AZ-2b     AZ-2c
  GW pod    GW pod    GW pod       GW pod    GW pod    GW pod
  (ASG)     (ASG)     (ASG)        (ASG)     (ASG)     (ASG)
```

**Layers of protection:**
- **Instance level:** Auto-scaling group replaces unhealthy pods automatically
- **AZ level:** Traffic shifts to healthy AZs if one data center fails
- **Regional level:** Global LB routes away from a failed region entirely

**Sizing rule:** If you need N instances for peak load, run N+M (e.g., need 3, run 5). This absorbs instance failures without degradation.

**Auto-scaling triggers:**
- Scale up: CPU > 60% or request count > threshold
- Scale down: CPU < 40%
- Scale out *before* hitting 100% — NLB target registration has 30-60s delay


---

## 8. Distributed Rate Limiting

### The Problem

10 gateway pods with per-pod counters = a client can send 10x the intended limit. Rate limiting must be globally consistent.

### Solution: Centralized Redis Cluster

- Redis is in-memory → single-digit millisecond latency
- All gateway instances share one Redis cluster
- Algorithm per request:
  1. Construct key: `ratelimit:{apiKey}:{timestamp_rounded_to_minute}`
  2. `INCR` the key, set `EXPIRE` (e.g., 60s) atomically
  3. If returned count > limit → return `429 Too Many Requests`
  4. Otherwise → process the request

### The Latency Problem

A Redis network call on every single request adds latency to the hot path. At 50K RPS this matters.

### Solution: Two-Tiered Hybrid Approach

```
Incoming Request
      │
      ▼
Local In-Memory Cache (LRU)
  ├── Client count < 80% of limit? → Allow immediately (no Redis call)
  └── Client count ≥ 80% of limit? → Promote to Redis check
                                          │
                                          ▼
                                   Redis (authoritative)
                                   ├── Count ≤ limit? → Allow
                                   └── Count > limit? → 429
```

- **99% of well-behaved clients:** Handled locally, zero Redis network call
- **1% approaching/exceeding limit:** Checked against Redis for accuracy
- Best of both worlds: speed for normal traffic, correctness for abusive traffic

### Redis as a Critical Dependency — The Tradeoff

Centralizing rate limiting in Redis creates a new SPOF. If Redis is slow, **every request** is slow.

**Mitigations:**
- Redis Cluster (not single node) for HA
- Monitor Redis latency as a first-class SLO
- Have a "break-glass" procedure: push config to gateway to skip Redis checks (disables rate limiting, restores availability)
- This is a deliberate tradeoff: consistency of rate limiting vs. availability of the platform


---

## 9. Zero-Downtime Config Changes

### The Old Way (Dangerous)

Edit a config file → restart the proxy process → all traffic drops during restart. A bug in config logic crashes the proxy.

### The Right Way: Dynamic Configuration + Canary Rollout

**Step 1: Dynamic Config Store**
- Gateway instances do NOT load config from static files at startup
- They pull from a dynamic store: **Consul, etcd, or AWS AppConfig**
- Modern gateways (Envoy, Kong, Traefik) support "hot reload" — apply new routing rules without restarting or dropping connections

**Step 2: Canary Rollout Process**

```
New config pushed to store
        │
        ▼
Apply to 1-5% of gateway fleet (canary group)
        │
        ▼
Monitor for 5-10 minutes:
  - Error rate (4xx, 5xx)
  - P99 latency
  - CPU usage
        │
   ┌────┴────┐
Healthy?   Degraded?
   │           │
   ▼           ▼
Expand to   AUTO-ROLLBACK:
25% → 50%   Push previous config
→ 100%      to config store.
            Canary instances
            revert immediately.
```

**Automated rollback trigger:** If error rate for canary group spikes by >5%, deployment pipeline automatically pushes the last known-good config. No human needed at 3 AM.

**Why this works:** The control plane pushes config to data planes. Rolling back is just pushing a different config version — same mechanism, same speed, no rebuild required.


---

## 10. Partner API BFF — Special Concerns

The Partner BFF is not a UI optimization tool. It is a **product** that other companies build on. Stability and predictability are the primary features.

### Comparison: Mobile BFF vs Partner BFF

| Concern | Mobile BFF | Partner BFF |
|---|---|---|
| Pace of change | Fast — daily/weekly for UI features | Slow & deliberate — stability is the feature |
| Contract | Loose — BFF and app often deploy in lockstep | Strict — formal OpenAPI/Swagger spec. Breaking it has business/legal consequences |
| Authentication | User-centric (JWTs) | Application-centric (API Keys per partner) |
| Rate limiting | Global/per-user abuse prevention | Per-partner contracted quotas (Gold: 100 req/s, Silver: 20 req/s) — this is a billing feature |
| Documentation | Internal, can be informal | Public developer portal — part of the product |
| Onboarding | N/A | Self-service: API key generation, docs, usage analytics |
| Versioning | Implicit, coordinated deploys | Explicit, versioned, announced months in advance |

### Additional Architectural Concerns for Partner BFF

- **SLA enforcement:** Partners have contractual uptime guarantees. The Partner BFF needs its own SLO tracking separate from internal BFFs.
- **Audit logging:** Every partner API call must be logged with partner identity for billing, debugging, and compliance.
- **Usage analytics dashboard:** Partners need self-service visibility into their own usage, quota consumption, and error rates.
- **Sandbox environment:** Partners need a safe environment to test integrations without hitting production data.


---

## 11. API Versioning & Deprecation Lifecycle

### Versioning Strategy: URL Path Versioning

**Chosen approach:** `/v1/orders`, `/v2/orders`

**Why URL path over header-based (`Accept: application/vnd.api.v1+json`):**
- Explicit and unambiguous — visible in browser, logs, curl commands
- Routing at gateway/load-balancer level is trivial (just match path prefix)
- Universally understood by developers of all experience levels
- REST purists prefer headers, but operational simplicity wins for public APIs

### Deprecation Lifecycle (The Right Way)

A deprecation is a product communication problem as much as a technical one.

**Phase 1: Announce (6-12 months ahead)**
- Publish deprecation schedule on developer portal, blog, changelog
- Direct email to all partners using the deprecated version
- Add `Deprecation` and `Sunset` headers to all v1 responses:
  ```
  Deprecation: true
  Sunset: Sat, 31 Dec 2025 23:59:59 GMT
  Link: <https://api.example.com/v2/orders>; rel="successor-version"
  ```

**Phase 2: Monitor & Identify Laggards**
- Log every request to v1 endpoints tagged by `apiKey`
- Build a dashboard: "Partners still on v1 and their request volume"
- Proactively reach out to high-volume v1 users

**Phase 3: Brownouts (1-2 months ahead)**
- Temporarily disable v1 for short scheduled windows (e.g., 10 min every Tuesday at 2 AM)
- Return `503 Service Unavailable` with a body explaining the deprecation and migration guide
- This is highly effective — it forces action from teams who ignored the emails
- Announce brownout schedule in advance

**Phase 4: Direct Contact**
- Reach out personally to remaining laggards identified in monitoring
- Offer migration support if needed

**Phase 5: Sunset**
- Permanently disable v1
- Return `410 Gone` (not 404) — signals intentional, permanent removal
- `410` tells clients "stop trying, this is gone forever"


---

## 12. Retry Storm Protection

### Scenario: A partner's buggy client is hammering the system with retries

**Layer 1: Per-Partner Hard Rate Limit (Blast Radius Containment)**
- Gateway enforces quota by API key
- Partner exceeds limit → immediate `429 Too Many Requests`
- Blast radius is contained to that single partner — other partners unaffected

**Layer 2: Informative Response Headers**
```
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1735689600
```
- `Retry-After` tells the client exactly when to retry — well-behaved clients will back off
- `X-RateLimit-*` headers allow clients to proactively throttle before hitting the limit

**Layer 3: Circuit Breakers in the BFF**
- If the retry storm on a specific endpoint starts overloading a downstream service (e.g., `ItemService`)
- Circuit breaker in the Partner BFF trips
- BFF stops forwarding to `ItemService` for a recovery window
- Returns `503 Service Unavailable` immediately to the partner
- Protects downstream services from cascading failure

**Layer 4: Self-Service Observability**
- Partner's developer portal shows real-time usage graph and error rates
- When they see their request volume spike alongside `429`s, they self-diagnose
- Turns a support ticket into a self-service resolution
- Reduces on-call burden


---

## 13. Data Plane vs Control Plane

This is the most important architectural innovation in modern proxy/gateway infrastructure.

### The Separation

**Data Plane (e.g., Envoy)**
- The "muscle" — sits in the request path
- Highly optimized, stateless process
- Only job: proxy traffic as fast as possible based on given config
- Dumb by design — no business logic, no config generation

**Control Plane (e.g., Istio, custom Go service, Kong Admin API)**
- The "brain" — NOT in the request path
- Translates high-level intent ("route `/products` to `ProductService`") into low-level Envoy xDS config
- Talks to service discovery, certificate stores, policy engines
- Pushes config to data planes via xDS APIs (gRPC streaming)

### What Breaks If You Collapse Them

| Problem | Impact |
|---|---|
| High-risk deployments | A bug in config logic crashes the proxy process — config change takes down traffic |
| No dynamicism | Every config change requires a restart/reload — slow, risky |
| Can't scale independently | Config generation is heavier than proxying. Collapsed = run 100 bloated pods instead of 100 lean data planes + 3 control planes |
| Security boundary violation | Data plane needs minimal permissions. Control plane needs to talk to service discovery, secrets, etc. Collapsing them gives the data plane excessive privileges |

### The Critical Property: Fail-Static

**If the control plane goes down, live traffic is completely unaffected.**

- Data planes receive config and load it into memory
- They operate independently of the control plane
- Control plane failure = you lose the ability to *change* config, not the ability to *serve* traffic
- This is a massive operational win: management plane failure ≠ revenue-generating traffic failure

### Practical Implication

You can take down Istio's control plane for maintenance, upgrades, or debugging without touching a single live request. This is why the separation matters.


---

## 14. Resilience Config Placement

Where do timeouts, retries, and circuit breakers live? This is a nuanced decision.

### Option 1: At the Edge Gateway

| Pros | Cons |
|---|---|
| Simple, centralized | Blind to east-west (service-to-service) traffic |
| Easy to implement | Too coarse — one timeout for all services is wrong |
| Good for north-south defaults | `UserService` needs 50ms timeout, `ReportingService` needs 2s |

### Option 2: In Each Service's Code (Libraries)

| Pros | Cons |
|---|---|
| No extra infrastructure | Version drift across 30 services |
| Teams have full control | Inconsistent implementations (Java Resilience4j vs Go's custom logic) |
| | No central visibility or enforcement |
| | Changing a timeout requires redeploying a service |
| | Operational nightmare at scale |

### Option 3: Service Mesh (Sidecar Proxy per Service)

| Pros | Cons |
|---|---|
| Config managed centrally by control plane | High complexity — major infrastructure investment |
| Enforced locally by sidecar (fine-grained, per-service) | Resource overhead (a proxy sidecar per pod) |
| Covers both north-south AND east-west traffic | Steep learning curve for the entire org |
| Language agnostic | |
| e.g., "BFF → ProductService: 3 retries, 100ms timeout" | |

### Recommended: Pragmatic Two-Tiered Evolution

**Phase 1 (Now): Gateway defaults**
- Define sane global defaults at the gateway for north-south traffic
- e.g., 30s max timeout, 2 retries with exponential backoff
- Immediate safety net, low complexity

**Phase 2 (As org matures): Service Mesh**
- Adopt Istio or Linkerd when east-west complexity justifies it
- Control plane provides central management
- Sidecars provide per-service enforcement
- Migrate incrementally — not a big-bang cutover

**The incremental migration path (staff-level detail):**
1. Start with the service mesh in "permissive" mode (observe, don't enforce)
2. Onboard one non-critical service, validate behavior
3. Gradually onboard services, starting with those with the most complex resilience needs
4. Only enforce mTLS and policies after all services are onboarded


---

## 15. Production Incident Playbook

### Scenario: 11 PM Friday. 40% error rate. P99 latency 8s. Downstream services healthy.

**Priority order: Mitigate → Diagnose → Communicate**

---

### Step 1: Triage (First 60 seconds)

- Check main dashboard: error rate 40%, P99 8s, affects ALL endpoints
- Confirm downstream health: CPU flat, internal error rates zero
- **Initial hypothesis:** Problem is between client and services — gateway or its dependencies
- **8-second latency is a clue:** Suspiciously round number → likely a timeout somewhere
- **Error type guess:** Probably `504 Gateway Timeout` or `503 Service Unavailable`

---

### Step 2: Mitigate — Stop the Bleeding (1-5 minutes)

**Action 1: Rollback the last change (do this first, ask questions later)**
- Check CI/CD dashboard or gateway config git history
- Was there a config push today? Even hours ago?
- Hit "redeploy previous version" — control plane pushes last known-good config to data planes
- Watch dashboard: does latency/error rate recover?
  - **Yes → crisis over.** Mark bad config as "do not deploy," open post-mortem ticket
  - **No → last change wasn't the cause, proceed**

**Action 2: Check gateway resource usage**
- Are gateway pods pegged at 100% CPU or memory?
- If yes → manually override auto-scaler, double the pod count
- Resource exhaustion from a traffic spike would cause exactly this symptom

**Action 3: Check Redis (rate limiting dependency)**
- Check Redis dashboard: is latency high? CPU at 100%?
- If Redis is slow → **every single request** is slow (it's in the hot path)
- **Break-glass procedure:** Push config to gateway to skip Redis checks
  - Effectively disables rate limiting temporarily
  - Sacrifices abuse protection for platform availability
  - Acceptable for a short window to restore service

---

### Step 3: Deeper Diagnosis (If still unresolved)

- **Check gateway logs:** What is the actual error? All `504`s confirms gateway is waiting and timing out
- **Check connection pool metrics:** Is the gateway exhausted of connections to downstream services?
  - Downstream services may look "healthy" but be slow to accept new connections
  - Connection pool exhaustion = gateway queues requests → timeout → 504
- **Check DNS:** `exec` into a gateway pod, `curl` a downstream service by internal DNS name
  - Can the pod resolve the name? DNS failures are rare but catastrophic and hard to spot

---

### How Earlier Decisions Helped or Hurt

**Decisions that HELPED:**

| Decision | How It Helped |
|---|---|
| Data plane / control plane split + canary deploys | Gave us the "rollback button" — fastest, safest path to resolution |
| Comprehensive observability | Separate dashboards for gateway, services, Redis — without this we're blind |
| Paved road for BFFs | Standard metrics/logs across all BFFs — no guessing how Mobile BFF logs differ from Web BFF |
| Multi-AZ/multi-region redundancy | System is degraded, not dead — regional failover is working |

**Decisions that HURT (or introduced risk):**

| Decision | The Risk |
|---|---|
| Centralized Redis for rate limiting | Created a new critical dependency in the hot path. Slow Redis = slow everything. This is the most likely cause of this specific incident. |

**The lesson:** Every architectural decision that improves one property (consistency of rate limiting) creates a new dependency that can fail. The investment in Redis observability is what makes it diagnosable. Design for failure, instrument everything.


---

## 16. Architectural Tradeoffs Summary

| Decision | What You Gain | What You Risk |
|---|---|---|
| Centralized gateway | Consistent security, observability, rate limiting | New Tier-0 SPOF — must be designed for HA |
| BFF per client type | Client-optimized payloads, team autonomy | More services to operate and maintain |
| Redis for distributed rate limiting | Global consistency across all gateway pods | Redis becomes a hot-path dependency — its latency is your latency |
| OPA for authorization | Consistent policies, no redeploy for policy changes | New sidecar dependency per service, Rego learning curve |
| URL path versioning | Operational simplicity, easy routing | URL "pollution," REST purists disagree |
| Data plane / control plane split | Fail-static behavior, independent scaling | More infrastructure components to operate |
| Service mesh (long-term) | Fine-grained resilience config, east-west coverage | High complexity, resource overhead, org learning curve |
| Paved road BFF ownership | Team autonomy + platform consistency | Platform team must stay connected to feature team needs |

---

## 17. Gaps to Sharpen (Staff-Level Delta)

These are the areas that separate senior from staff in this topic area.

### 1. Connection Pool Management

The gateway at 50K RPS with 30 downstream services has a serious connection management problem.

**Questions to have crisp answers for:**
- How many connections does each gateway pod maintain to each downstream service?
- What happens when a slow downstream service holds connections open? (Connection pool exhaustion → queue → timeout → 504)
- HTTP/2 multiplexing vs HTTP/1.1 connection pools — what's the difference and when does it matter?
  - HTTP/2: multiple requests over one connection (multiplexing) — fewer connections, but head-of-line blocking within a stream
  - HTTP/1.1: one request per connection — more connections needed, but simpler failure isolation

### 2. Service Mesh Migration Path

"Start with gateway, evolve to service mesh" is correct but hand-wavy. The transition is the hard part.

**The incremental path:**
1. Deploy mesh in observe-only mode (no policy enforcement, just metrics)
2. Onboard one non-critical service — validate sidecar behavior, resource overhead
3. Gradually onboard services by criticality (least critical first)
4. Enable mTLS in permissive mode (accepts both mTLS and plaintext)
5. Switch to strict mTLS only after all services are onboarded
6. Never do a big-bang cutover

### 3. BFF Distributed Tracing for Fan-out Calls

The BFF makes 8 parallel calls per request. If the home screen is slow, which of the 8 calls was the bottleneck?

**What's needed:**
- Distributed tracing with per-span timing across all parallel calls
- BFF must propagate trace context (`traceparent` header per W3C Trace Context spec) on every outbound call
- Each downstream service adds its own span to the trace
- Result: a waterfall view showing exactly which service added latency
- This is a first-class BFF design concern, not a general observability footnote

### 4. Thundering Herd on Gateway Startup

When scaling out 20 new gateway pods simultaneously during an incident:
- All 20 pods hit the control plane for config at the same time
- All 20 pods hit Redis to warm their local rate-limit cache
- All 20 pods try to establish connection pools to 30 services simultaneously

**Mitigations:**
- Kubernetes readiness probes: pod doesn't receive traffic until it's fully initialized
- Staggered startup: use `maxSurge` and `maxUnavailable` in rolling deploy config to limit simultaneous pod starts
- Control plane should handle burst config requests gracefully (rate limit or queue them)
- Pre-warm connection pools before marking pod as ready

### 5. The "Lying Health Check" Equivalent for the Gateway

A gateway pod can be "healthy" (process running, responding to `/health`) but functionally broken (Redis connection lost, config store unreachable, connection pool exhausted).

**Deep health check for gateway should verify:**
- Can reach Redis (rate limiting dependency)
- Can reach config store (Consul/etcd)
- Can establish a connection to at least one downstream service
- Connection pool has available connections

---

## Quick Reference: Key Numbers & Rules of Thumb

| Metric | Rule |
|---|---|
| Gateway instances | N+M (need 3 for peak → run 5) |
| Auto-scale trigger | CPU > 60% scale up, < 40% scale down |
| NLB registration delay | 30-60s — scale out before hitting 100% |
| Canary rollout size | 1-5% of fleet |
| Canary observation window | 5-10 minutes before expanding |
| Auto-rollback trigger | Error rate spike > 5% on canary |
| API deprecation notice | 6-12 months ahead |
| Brownout window | 10 min, weekly, 1-2 months before sunset |
| Sunset response code | 410 Gone (not 404) |
| Redis rate limit key pattern | `ratelimit:{apiKey}:{timestamp_rounded_to_minute}` |
| Local cache promotion threshold | 80% of rate limit → promote to Redis check |

