# Load Balancing & Service Discovery

A comprehensive system design for distributing traffic evenly across services and enabling dynamic service discovery in a polyglot microservices architecture.

---

## Table of Contents

1. [Problem Statement](#problem-statement)
2. [Requirements](#requirements)
3. [Clarifying Questions & Context](#clarifying-questions--context)
4. [High-Level Architecture](#high-level-architecture)
5. [Core Design Decision: Server-Side vs Client-Side Discovery](#core-design-decision-server-side-vs-client-side-discovery)
6. [Component Deep Dives](#component-deep-dives)
   - [Health Checking](#1-health-checking-from-12-minutes-to-seconds)
   - [Service Registry (Consul)](#2-service-registry-consul---the-source-of-truth)
   - [Load Balancing Layer (Traefik)](#3-load-balancing-layer-traefik---the-traffic-cop)
7. [Hardening & Fault Tolerance](#hardening--fault-tolerance)
   - [Consul Failure Modes](#consul-failure-modes)
   - [Capacity Planning](#capacity-planning-for-400k-rpm)
   - [Zero-Downtime Deployments](#zero-downtime-rolling-deployments)
8. [Production Incident: The Lying Health Check](#production-incident-the-lying-health-check)
9. [Evolutionary Architecture](#evolutionary-architecture)
10. [Interview Assessment & Takeaways](#interview-assessment--takeaways)

---

## Problem Statement

A fast-growing food delivery company (~15 microservices) handling ~5,000 orders/minute at peak, expanding to 3 new countries with expected 10x growth.

**Current Pain Points:**
- Services communicate via **hardcoded IP addresses** in config files
- Deployments require **manual config updates**
- A payment service instance went down and kept receiving traffic for **12 minutes** before someone noticed
- Orders were failing silently

**Design Goals:**
1. No more hardcoded IPs — services should find each other dynamically
2. Traffic distributed intelligently across healthy instances
3. Handle instance failures gracefully with minimal impact on live traffic
4. Scale from 5K orders/min to 50K orders/min

---

## Requirements

### Functional Requirements

| Requirement | Specification |
|-------------|---------------|
| **Dynamic Discovery** | Services find each other without hardcoded IPs |
| **Load Distribution** | Traffic evenly distributed across healthy instances |
| **Health Detection** | Failing instances removed from rotation in seconds |
| **Zero-Downtime Deploys** | Rolling deployments with no dropped requests |
| **Multi-Protocol** | Support HTTP/REST and gRPC traffic |

### Non-Functional Requirements

| Requirement | Specification |
|-------------|---------------|
| **Scale** | 50K orders/min (10x growth), ~400K internal RPM |
| **Latency** | Sub-millisecond overhead for proxy hop |
| **Availability** | System remains operational during partial failures |
| **Polyglot Support** | Java, Go, Node.js, Python services |
| **Future-Proof** | Align with planned Kubernetes migration |

---

## Clarifying Questions & Context

Before designing, the following context was established:

| Question | Answer | Impact on Design |
|----------|--------|-----------------|
| Can services scale independently? | Yes, containerized (Docker), moving toward K8s | Need dynamic registration/deregistration |
| Infrastructure? | AWS, hybrid EC2/ECS (Fargate), K8s 6+ months out | Solution must work in hybrid world |
| Tech stack? | Polyglot: Java (Spring Boot), Go, Node.js, Python | Client-side discovery becomes expensive |
| Communication protocols? | Mostly HTTP/REST, some gRPC, Kafka for async | Proxy must handle multiple protocols |
| Observability? | Basic CloudWatch, partial Prometheus, ELK (unstructured) | Need to integrate with and improve observability |
| Team maturity? | Small team, inconsistent alerting | Favor simplicity over raw power |

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        MICROSERVICES (15+ services)                         │
│   Java (Spring Boot), Go, Node.js, Python                                  │
│   Each with Consul Agent sidecar + /health/ready endpoint                  │
└─────────────────────────────┬───────────────────────────────────────────────┘
                              │ Self-Registration via localhost:8500
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     CONSUL CLUSTER (5 nodes, multi-AZ)                     │
│   • Raft consensus for leader election                                      │
│   • Service catalog + health state                                          │
│   • Gossip protocol for state propagation                                   │
│   • TTL heartbeats + active HTTP health checks                              │
└─────────────────────────────┬───────────────────────────────────────────────┘
                              │ Blocking queries (long-poll, near real-time)
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     TRAEFIK FLEET (3+ instances, ASG)                       │
│   • Auto-discovers backends from Consul                                     │
│   • Weighted Round Robin (default) / Least Connections                      │
│   • Passive health checks (outlier detection)                               │
│   • Circuit breaking, retries, timeouts                                     │
└─────────────────────────────┬───────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│              AWS NETWORK LOAD BALANCER (NLB)                                │
│   • Single stable entry point for the VPC                                   │
│   • Hyper-scalable, managed, millions of RPS                                │
│   • Cross-AZ distribution                                                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Core Design Decision: Server-Side vs Client-Side Discovery

### Decision: Server-Side Discovery

**Decision Framework:**

| Criterion | Weight | Client-Side | Server-Side |
|-----------|--------|-------------|-------------|
| Speed of Implementation | High | ❌ Build 4 client libs | ✅ One proxy fleet |
| Operational Simplicity | High | ❌ Distributed complexity | ✅ Centralized config |
| Cross-Team Consistency | High | ❌ Per-language behavior | ✅ Uniform policies |
| Future-Proofing (K8s) | Medium | ❌ Must rip out later | ✅ Same mental model |
| Latency | Low | ✅ No extra hop | ⚠️ Sub-ms overhead |
| Decentralized Failure | Low | ✅ Warm cache survives | ⚠️ Proxy is dependency |

### Client-Side Discovery Analysis (The "Netflix/Eureka" Model)

**How it works:** A smart client library in each microservice queries the service registry, caches healthy instances, performs client-side load balancing, and handles retries/circuit breaking.

**Pros:**
- **Performance:** Eliminates the extra network hop of a central proxy — lowest possible latency
- **Decentralized Failure:** Clients with warm cache can continue operating briefly if registry fails

**Cons (Deal-breakers for our context):**

1. **The Polyglot Nightmare:**
   - Netflix could invest in one library (Ribbon) because they were a Java shop
   - We need 4 separate, feature-equivalent libraries (Java, Go, Node, Python)
   - A bug in the Python retry logic but not the Go one = inconsistent, hard-to-debug behavior
   - Massive cognitive load and maintenance burden

2. **Slows Innovation:**
   - New language (e.g., Rust) requires building a compatible discovery library first
   - Global changes (TLS cert strategy) require coordinated deployment of all 15 services
   - With a central proxy, update in one place

3. **Blurs Responsibilities:**
   - Application developers forced to own network-level plumbing
   - Debugging becomes complex: is it the app code, the discovery library, the network, or the target service?

### Server-Side Discovery Analysis (The "Proxy" Model)

**How it works:** Applications are "dumb" — they send traffic to a stable proxy endpoint. The proxy handles all discovery, load balancing, and health-checking logic.

**Pros (Perfect for our context):**

1. **Language Agnostic:**
   - Developers change `payment.service.url` from hardcoded IP to `http://internal-proxy/payment-service`
   - Zero code changes, zero library additions
   - All routing, retries, timeout policies configured centrally

2. **Fast to Implement:**
   - Infrastructure team focuses on one piece: registry (Consul) + proxy fleet (Traefik)
   - Roll out incrementally, prove value with a few services first

3. **Excellent K8s Path:**
   - Conceptually identical to K8s Ingress Controller / Gateway API
   - Proxy = Ingress, Service registry = etcd backend for K8s Services
   - Teaching the org this mental model now paves the way for seamless migration

**Cons (Mitigated):**

| Con | Mitigation |
|-----|-----------|
| Extra network hop (~sub-ms) | Acceptable for food delivery; not HFT |
| Central point of failure | HA multi-node fleet behind AWS NLB |

---

## Component Deep Dives

### 1. Health Checking: From 12 Minutes to Seconds

The core of the outage was a lack of timely, automated health information. Health checking is the central nervous system of the platform.

#### Ownership Model

The **Service Registry (Consul) owns the source-of-truth for health**. The actual check is performed by a Consul agent running locally alongside each service instance. The proxy is a *consumer* of this health information, not a generator of it.

**Why not have the proxy check backends directly?**
Having each proxy instance individually check every backend creates a massive N×M polling storm. Consul centralizes this efficiently.

#### Health Check Configuration

Each microservice exposes a `GET /health/ready` endpoint. The local Consul agent is configured with:

```json
{
  "check": {
    "id": "payment-service-readiness",
    "name": "Payment Service Readiness Check",
    "http": "http://localhost:8080/health/ready",
    "interval": "10s",
    "timeout": "2s",
    "deregister_critical_service_after": "1m"
  }
}
```

#### The Detection Loop

```
Every 10s: Consul Agent → GET http://localhost:8080/health/ready

  200 OK → Service remains "passing" (no action)

  Non-2xx / Timeout (2s) → Agent marks check "critical"
    → Notifies Consul server cluster
    → Gossip protocol propagates in milliseconds
    → Traefik's blocking query returns immediately
    → Failing instance removed from routing pool
```

**Total detection time:** `interval (10s) + timeout (2s) + propagation (~1s) ≈ 13 seconds` (vs. 12 minutes before)

The `deregister_critical_service_after: 1m` is a safety net to remove "ghost" instances from the registry entirely.

#### Liveness vs. Readiness: A Critical Distinction

| Check | Purpose | Who Uses It | Failure Means |
|-------|---------|-------------|---------------|
| `/health/live` | "Is the process running?" | Orchestrator (ECS/K8s) | Kill and replace the container |
| `/health/ready` | "Can it serve traffic?" | Consul → Traefik | Remove from load balancer pool |

A service can be **live but not ready** when:
- Still starting up (warming caches, connecting to DB)
- Shutting down gracefully (draining connections)
- Temporarily overloaded

Using readiness checks prevents traffic to instances that are starting up or degraded — key for zero-downtime deployments.

---

### 2. Service Registry (Consul) — The Source of Truth

#### Registration Pattern: Self-Registration with Local Agent

The service itself is the most reliable source of information about its own state.

**For EC2 Instances:**
- Base AMI includes the Consul agent
- Startup script registers with local agent at `http://localhost:8500/v1/agent/service/register`

**For ECS Tasks:**
- Task Definition includes two containers: application + Consul agent sidecar
- Shared network namespace — application registers via `localhost:8500`
- Agent lifecycle tied directly to task lifecycle

#### Self-Registration vs. External Registration

| Aspect | Self-Registration (Chosen) | External Registration (Rejected) |
|--------|---------------------------|----------------------------------|
| Speed | Fast — announces when actually ready | Slow — polls AWS API, always lagging |
| Accuracy | Knows if *application* is ready | Only knows if *instance* is running |
| Coupling | Requires lightweight agent per host | Requires central god-like component |
| Complexity | Decentralized, simple per-service | Centralized, complex, brittle |
| Responsibility | Aligns responsibility with control | Separates knowledge from action |

#### Handling Staleness: TTL Heartbeating

Along with active HTTP health checks, a TTL check provides a second layer:

```
Consul Agent → heartbeat packet → Consul Server (every 10s)

If heartbeats stop (host vanishes, agent crashes, network partition):
  → TTL expires (30-60s)
  → Consul marks service as failed
```

This catches failure modes that active health checks miss, like complete network partitions where the agent can't even report failure.

---

### 3. Load Balancing Layer (Traefik) — The Traffic Cop

#### Why Traefik?

| Proxy | Assessment |
|-------|-----------|
| **Envoy** | Most powerful, but complex configuration (xDS APIs). Steep learning curve for a small team. |
| **NGINX (OSS)** | Requires manual config reloads for new backends. Non-starter for dynamic discovery. |
| **NGINX Plus** | Can do it, but commercial product. |
| **Traefik** | ✅ Built for dynamic configuration from providers like Consul. Easy to configure, great observability, right balance of power and simplicity. |

#### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    AWS Network Load Balancer                     │
│              (Single stable entry point for VPC)                │
└──────────┬──────────────────┬──────────────────┬────────────────┘
           │                  │                  │
           ▼                  ▼                  ▼
    ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
    │  Traefik #1  │   │  Traefik #2  │   │  Traefik #3  │
    │  (AZ-1a)     │   │  (AZ-1b)     │   │  (AZ-1c)     │
    └──────────────┘   └──────────────┘   └──────────────┘
           │                  │                  │
           └──────────────────┼──────────────────┘
                              │ Blocking queries
                              ▼
                    ┌──────────────────┐
                    │  Consul Cluster  │
                    └──────────────────┘
```

- **HA Fleet:** 3+ Traefik instances in an Auto Scaling Group, spread across 3 AZs
- **NLB:** Provides a single, static entry point for the whole VPC

#### Discovery Mechanism: Blocking Queries (Not Polling)

Traefik does **not** poll Consul. It uses an efficient **blocking query** (long-poll) mechanism:

```
1. Traefik → GET /v1/health/service/payment-service?wait=5m&index=42
2. Consul holds connection open
3. When health status or instance count changes:
   → Consul immediately responds with new healthy backend list
   → Closes connection
4. Traefik updates internal routing table
5. Traefik opens new blocking query (repeat)
```

This is near real-time and extremely efficient — no wasted polling cycles.

#### Load Balancing Algorithms

| Algorithm | When to Use | Our Default |
|-----------|-------------|-------------|
| **Weighted Round Robin** | Stateless REST services, equal instance capacity | ✅ Default |
| **Least Connections** | Services with uneven request costs (e.g., reporting) | Per-service override |
| **P2C (Power of Two Choices)** | Advanced: picks 2 random backends, sends to less loaded one | Future evolution |

All algorithm changes are configured declaratively per-service in Traefik — no application code changes needed.

---

## Hardening & Fault Tolerance

### Consul Failure Modes

Consul is the brain of the system. If it fails, the entire body is affected. The key insight: **we haven't eliminated a SPOF; we've moved it to something designed to be fault-tolerant.**

#### Scenario A: Leader Fails

**Context:** Consul uses Raft consensus. One server is leader (handles writes + consistent reads). Leader failure triggers election (~5-15 seconds).

**Blast Radius During Election:**
- **Writes fail:** No new registrations, de-registrations, or health status updates
- **Reads succeed (stale):** Traefik can still read from followers. Consul allows stale reads by default.

**Mitigations:**

| Mitigation | How It Helps |
|-----------|-------------|
| **Stale Reads** | Traefik routes based on last known-good list. Slightly less accurate for ~15s, but available. |
| **Application Retries** | If routed to a now-dead instance, retry with exponential backoff hits a healthy one. |
| **5-Node Cluster** | Tolerates 2 simultaneous node failures while maintaining quorum. |
| **Multi-AZ Spread** | Nodes across `us-east-1a`, `1b`, `1c`. Single AZ failure = single node failure. |

**Impact:** Small, temporary degradation (slightly stale routing). Not a catastrophic failure. System remains **available**.

#### Scenario B: Network Partition (Split Brain)

**Context:** Partition splits 5-node cluster into groups of 3 and 2.

**Behavior:**
- **Group of 3:** Maintains quorum `((5/2)+1 = 3)`, elects leader, operates normally as source of truth
- **Group of 2:** Loses quorum, cannot elect leader, rejects all writes — effectively offline for control plane

**Mitigations:**

| Mitigation | How It Helps |
|-----------|-------------|
| **Raft Quorum** | Prevents minority partition from making decisions — no true split brain |
| **Fail Safe Mode** | Agents/proxies on minority side continue using last known-good config |
| **Multi-AZ Deployment** | Single AZ failure is just a single-node failure for the cluster |

**Key Insight:** We accept a small, bounded risk of temporary data staleness in exchange for massive gains in automation and availability. The system is architected to be **resilient in the face of temporary registry unavailability**.

---

### Capacity Planning for 400K RPM

#### Back-of-Envelope Calculation

| Parameter | Value | Source |
|-----------|-------|--------|
| Target RPM | 400,000 internal requests/min | 50K orders × ~8 internal calls |
| Target RPS | ~6,700 requests/sec | 400K ÷ 60 |
| Proxy | Traefik (Go, efficient network I/O) | Design choice |
| Avg response time | 50ms | Assumption |
| Instance type | `c5.large` (2 vCPU, 4 GiB RAM) | CPU-optimized |

#### Bottleneck Analysis

| Resource | Assessment |
|----------|-----------|
| **CPU** | Primary bottleneck. TLS termination, request parsing, routing, middleware. |
| **Memory** | Scales with active connections + routing table. ~15 services, few hundred instances = small table. Not a concern. |
| **Network I/O** | `c5.large` has up to 10 Gbps. More than enough. |
| **Connection Tracking** | Linux `conntrack` table can bottleneck. Tune kernel for large concurrent connection counts. |

#### Capacity Math

```
Conservative estimate: 5,000 RPS per c5.large (with TLS + routing)
Peak target: 6,700 RPS
Minimum instances: 6700 / 5000 = 1.34 → round up

For redundancy + burst handling:
  Minimum: 3 instances (non-negotiable)
  Auto-scale trigger: CPU > 60% → scale up, CPU < 40% → scale down
```

#### Traefik Fleet Configuration

| Parameter | Value |
|-----------|-------|
| ASG Min | 3 |
| ASG Max | 10 |
| Instance Type | `c5.large` |
| AZ Spread | 3 AZs |
| Scale-Up Trigger | Avg CPU > 60% |
| Scale-Down Trigger | Avg CPU < 40% |

#### NLB Considerations

- **Not a bottleneck:** AWS NLB is a distributed, managed service handling millions of RPS
- **Gotcha:** NLB target registration has 30-60s delay for health checks to pass. Scaling policies must scale out *before* reaching 100% capacity.

---

### Zero-Downtime Rolling Deployments

The acid test of the entire system. Deploying `payment-service-v1.2` (replacing 10 instances) during Friday evening peak.

#### Step-by-Step Instance Lifecycle

```
INITIAL STATE:
  10 × payment-service-v1.1 running
  Consul: 10 healthy instances
  Traefik: 10 IPs in routing pool, traffic evenly distributed
```

**Step 1: Spin Up New Instance**
```
ECS starts payment-service-v1.2 task
  → Container boots, /health/ready returns 503 (still initializing)
  → Registers with Consul agent (readiness check defined)
  → Consul marks instance as "critical"
  → Traefik sees "critical" → does NOT add to routing pool
  → Zero production traffic to new instance
```

**Step 2: New Instance Becomes Ready**
```
Application finishes startup (DB connected, caches warmed)
  → /health/ready returns 200 OK
  → Consul agent detects on next poll (within 10s)
  → Reports "passing" to Consul server
  → Traefik's blocking query returns immediately
  → New instance added to routing pool
  → Receives 1/11th of traffic
```

**Step 3: Begin Shutting Down Old Instance**
```
ECS sends SIGTERM to v1.1 container
  → Application traps SIGTERM, begins graceful shutdown
  → FIRST ACTION: /health/ready starts returning 503
  → Within 10s, Consul agent detects, reports "critical"
  → Traefik removes old instance from routing pool
  → No NEW requests sent to this instance
```

**Step 4: Connection Draining**
```
Old instance has no new requests but may have active in-flight requests
  → Waits for grace period (30s) for existing requests to complete
  → After timer expires (or all connections close), application exits
  → ECS marks task as terminated
```

**Repeat** for all 10 instances. Result: **zero dropped requests**.

```
FINAL STATE:
  10 × payment-service-v1.2 running
  Consul: 10 healthy instances (all v1.2)
  Traefik: 10 new IPs in routing pool
```

---

## Production Incident: The Lying Health Check

### The Scenario

3 months after go-live. Order service experiencing **30% error rates** for requests to the restaurant service. All 6 restaurant service instances show as **healthy in Consul**. CPU and memory look fine. No recent deployments.

Traefik metrics reveal: **one of 6 instances returning 500s on ~80% of its requests**, but its `/health/ready` endpoint still returns 200.

### Root Cause

**The health check was a lie.** It checked if the process was running, not if it was *functional*.

Possible causes:
- Exhausted database connection pool
- Lost credentials to a downstream dependency
- Loaded corrupted configuration data
- Subtle bug affecting a specific code path

### Part 1: Immediate Triage (On-Call Response)

**Goal:** Stop the bleeding in under a minute.

```bash
# Place the misbehaving instance into Maintenance Mode
curl --request PUT \
  --data '{"enable": true, "reason": "Instance returning high 5xx rate"}' \
  http://consul.mycompany.local:8500/v1/agent/maintenance/service_id:<restaurant-service-instance-123>
```

**Effect:**
- Consul immediately marks instance as unhealthy
- Propagates to Traefik
- Traefik pulls instance from routing pool
- Error rate drops to near-zero within seconds
- Bad instance quarantined for safe post-mortem analysis

### Part 2: Short-Term Fix — Deeper Health Checks

Establish a new contract: `/health/ready` must be a **shallow integration test** of critical dependencies.

```
function handle_health_ready_request():
    // 1. Check database connectivity
    try:
        connection = db_pool.getConnection(timeout=100ms)
        connection.execute("SELECT 1")
        connection.release()
    except:
        return 503 Service Unavailable ("DB connection failed")

    // 2. Check critical cache health
    if not cache.isHealthy():
        return 503 Service Unavailable ("Cache unhealthy")

    // 3. Check other critical dependencies
    if not secrets_manager.isAccessible():
        return 503 Service Unavailable ("Secrets inaccessible")

    // All checks pass — truly ready
    return 200 OK
```

### Part 3: Long-Term Evolution — Self-Healing System

Even deeper health checks can't anticipate every novel failure. Build a safety net for the unknown.

#### Evolution 1: Passive Health Checks (Outlier Detection)

Configure Traefik to monitor actual response traffic:

```
Rule: If any backend returns 5 consecutive 5xx responses
  → Eject from pool for 60 seconds
```

**Two layers of defense:**

| Layer | Mechanism | Detects |
|-------|-----------|---------|
| **Active (Consul)** | Service reports its own health | Known failure modes (DB down, cache unhealthy) |
| **Passive (Traefik)** | Proxy observes real traffic | Unknown failure modes (subtle bugs, partial failures) |

#### Evolution 2: Adaptive Load Balancing

| Strategy | How It Works | Benefit |
|----------|-------------|---------|
| **P2C (Power of Two Choices)** | Pick 2 random backends, send to less loaded one | Naturally favors healthier, faster instances |
| **Least Connections** | Route to instance with fewest active connections | Sheds load from struggling instances |

#### Evolution 3: Circuit Breaking

Configure per-service circuit breakers in Traefik:

```
If overall error rate for restaurant-service > 50% for 30s:
  → Circuit trips
  → Traefik returns 503 immediately to callers
  → Fails fast, protects upstream from cascading failures
  → Gives downstream time to recover
```

**The evolution path:** From a system that simply *reports* health → one that **observes, learns, and reacts** to real-time conditions.

---

## Evolutionary Architecture

### Phase 1: Immediate (Weeks 1-4)
- Deploy Consul cluster (5 nodes, multi-AZ)
- Deploy Traefik fleet (3+ instances behind NLB)
- Migrate 2-3 pilot services from hardcoded IPs
- Implement basic `/health/ready` endpoints
- Set up Consul + Traefik dashboards in Grafana

### Phase 2: Rollout (Months 2-3)
- Migrate all 15 services to Consul registration
- Implement deep health checks (dependency-aware)
- Enable passive health checks / outlier detection in Traefik
- Standardize graceful shutdown (SIGTERM handling) across all services
- Integrate alerting on service discovery events

### Phase 3: Maturity (Months 4-6)
- Implement circuit breaking for critical service paths
- Explore adaptive load balancing (P2C / Least Connections)
- Prepare for Kubernetes migration (Consul Connect → K8s Services)
- Add canary deployment support via weighted routing

---

## Interview Assessment & Takeaways

### What Went Well

| Area | Strength |
|------|----------|
| **Clarifying Questions** | Thorough context gathering before designing. Infrastructure, tech stack, protocols, observability, team maturity. |
| **Decision Framework** | Built explicit criteria (speed, simplicity, consistency, future-proofing) and evaluated both patterns honestly. |
| **Netflix Counterpoint** | Correctly identified that Netflix's client-side success was context-dependent (Java monoglot) and doesn't apply to a polyglot shop. |
| **Operational Depth** | Raft consensus during partitions, stale reads as deliberate availability trade-off, SIGTERM handling, connection draining, conntrack tuning. |
| **Quantitative Reasoning** | Detection time math (interval + timeout + propagation ≈ 13s), capacity planning with conservative assumptions. |
| **Curveball Response** | Three-part structure (triage → fix → evolve) for the lying health check scenario. Traced design gap and proposed layered defense. |

### Areas to Sharpen

| Area | Improvement |
|------|------------|
| **Conversational Flow** | Pause during trade-off discussions and ask "does this trade-off make sense for your context?" Turns monologue into dialogue. |
| **Answer Length** | Answers are comprehensive but long. In a 45-min interview, identify the 20% that delivers 80% of signal. Offer to go deeper rather than defaulting to it. |
| **Observability Earlier** | Prometheus and ELK were mentioned in clarifying phase but not woven into core design until the curveball. Instrument the proxy layer, set up SLO-based alerts, and build dashboards from day one. |

### Key Concepts to Remember

1. **Server-side discovery** is the right default for polyglot, fast-growing teams with immature infrastructure
2. **Health checks must test functionality, not just liveness** — a process that's running but broken is worse than one that's down
3. **Stale reads are a feature, not a bug** — they keep the system available during registry hiccups
4. **Two layers of health detection** (active from registry + passive from proxy) catch both known and unknown failure modes
5. **Zero-downtime deploys** require coordination of readiness checks, SIGTERM handling, and connection draining
6. **Consul failure is bounded** — the system degrades gracefully (stale routing) rather than failing catastrophically
7. **Capacity plan conservatively** — assume lower per-instance throughput, scale out before hitting limits
8. **AI CHAT** — [ref]https://chat.swiggy.cloud/s/6e3a266d-8ed3-40d8-8034-2cae82ac32f5