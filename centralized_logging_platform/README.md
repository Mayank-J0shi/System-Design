# Centralized Logging & Analytics Platform

A comprehensive system design for ingesting, indexing, querying, and alerting on logs, metrics, and tracing data from thousands of microservices.

---

## Table of Contents

1. [Problem Statement](#problem-statement)
2. [Requirements](#requirements)
3. [High-Level Architecture](#high-level-architecture)
4. [Component Deep Dives](#component-deep-dives)
   - [Ingestion Layer](#1-ingestion-layer)
   - [Buffering Layer (Kafka)](#2-buffering-layer-kafka)
   - [Storage Layer (Elasticsearch)](#3-storage-layer-elasticsearch)
   - [Alerting System (Apache Flink)](#4-alerting-system-apache-flink)
   - [Query Layer](#5-query-layer)
5. [Data Flow Examples](#data-flow-examples)
6. [Scalability & Bottlenecks](#scalability--bottlenecks)
7. [Trade-offs & Design Decisions](#trade-offs--design-decisions)
8. [Production Considerations](#production-considerations)
9. [v1 vs v2 Roadmap](#v1-vs-v2-roadmap)

---

## Problem Statement

A large e-commerce company with **thousands of microservices** needs a centralized observability platform. Currently, each team manages their own logging, creating challenges for:

- **Debugging:** Can't trace issues across service boundaries
- **Incident Response:** No single pane of glass during outages
- **Compliance:** Audit logs scattered across systems

### Core Requirements

1. **Ingest** application logs, metrics, and tracing data from all services
2. **Index** data for fast, filtered searches
3. **Query** in near real-time (e.g., "Show me all ERROR logs from `payment-service` in the last hour")
4. **Alert** when error rates exceed thresholds

### Scale

- **5 TB of log data per day** (steady state)
- **3-5x peaks** during holiday shopping (Black Friday, Cyber Monday)
- **Thousands of microservices** in multiple languages (Java, Python, Go, Node.js)

---

## Requirements

### Functional Requirements

| Requirement | Specification |
|-------------|---------------|
| **Data Types** | Logs, Metrics, Traces |
| **Query Types** | Filtered searches by service, log level, timestamp |
| **Alerting** | Threshold-based alerts (e.g., 5xx error rate > 100/min) |
| **Searchable Latency** | Logs searchable within 5-10 seconds |
| **Retention** | 3 days |

### Non-Functional Requirements

| Requirement | Specification |
|-------------|---------------|
| **Reliability** | At-least-once delivery (no log loss) |
| **Scalability** | Handle 5 TB/day with 3-5x peak capacity |
| **Resilience** | Noisy neighbor protection (one service can't overwhelm the system) |
| **Availability** | System remains operational during partial failures |
| **Access Control** | Open across teams (no multi-tenancy isolation required) |

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           MICROSERVICES (thousands)                         │
│   Java, Python, Go, Node.js services                                        │
│   Using OpenTelemetry SDK → logs, metrics, traces                          │
└─────────────────────────────┬───────────────────────────────────────────────┘
                              │ OTLP (localhost:4317)
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    OTEL COLLECTOR (DaemonSet per node)                      │
│   • Receives logs/metrics/traces via OTLP                                   │
│   • Batches & compresses                                                    │
│   • Enriches with metadata (pod, namespace, region)                         │
│   • Buffers locally if downstream unavailable                               │
└─────────────────────────────┬───────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              KAFKA CLUSTER                                  │
│   Topics: logs, metrics, traces                                             │
│   • Decouples producers from consumers                                      │
│   • Handles burst traffic                                                   │
│   • Provides durability & replayability                                     │
│   • Multiple consumer groups                                                │
└──────────┬──────────────────────────────────┬───────────────────────────────┘
           │                                  │
           │ Consumer Group: alerting         │ Consumer Group: indexing
           ▼                                  ▼
┌─────────────────────────┐      ┌─────────────────────────────────────────────┐
│     APACHE FLINK        │      │           ELASTICSEARCH CLUSTER             │
│   (Alerting Pipeline)   │      │                                             │
│                         │      │   Index pattern: logs-{service}-{date}      │
│ • Sliding windows       │      │   • 10-12 hot data nodes                    │
│ • Per-service 5xx count │      │   • ~40GB per shard, rollover-based ILM     │
│ • RocksDB state         │      │   • Delete after 3 days                     │
│ • Watermarks            │      │                                             │
└───────────┬─────────────┘      └──────────────────┬──────────────────────────┘
            │                                       │
            ▼                                       ▼
┌─────────────────────────┐             ┌─────────────────────────┐
│   alerts Kafka topic    │             │   KIBANA / GRAFANA      │
└───────────┬─────────────┘             │   Developer Query UI    │
            │                           └─────────────────────────┘
            ▼
┌─────────────────────────┐
│   ALERT DISPATCHER      │
│   • Deduplication       │
│   • Cooldowns           │
│   • Multi-channel       │
└─────┬─────────┬─────────┘
      │         │
      ▼         ▼
 PagerDuty   Slack
```

---

## Component Deep Dives

### 1. Ingestion Layer

The ingestion layer is responsible for getting observability data from thousands of microservices into the central platform without impacting application performance or reliability.

#### The Problem with Direct Approaches

**Option A: Direct HTTP API Calls**
```
Service → HTTP POST → Ingestion API → Kafka
```

Issues:
- Network hop for every log batch
- If API is slow/down, what happens to the service? Block? Retry? Drop?
- Every service needs retry logic, timeout handling, circuit breakers
- Tight coupling to observability infrastructure
- Cascading failures possible during incidents

**Option B: Direct Kafka Push**
```
Service → Kafka Producer → Kafka
```

Issues:
- Kafka clients are heavy (connection management, partitioning, batching)
- Client quality varies across languages
- Every service tightly coupled to Kafka cluster
- Hard to change infrastructure later

#### The Solution: Agent/Collector Pattern

```
┌──────────────────────────────────────────────────────────────────┐
│                         Kubernetes Node                          │
│                                                                  │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│   │ Pod A        │    │ Pod B        │    │ Pod C        │      │
│   │ Java Service │    │ Python Svc   │    │ Node.js Svc  │      │
│   │   (OTLP)     │    │   (OTLP)     │    │   (OTLP)     │      │
│   └──────┬───────┘    └──────┬───────┘    └──────┬───────┘      │
│          │                   │                   │               │
│          └───────────────────┼───────────────────┘               │
│                              │ localhost:4317                    │
│                              ▼                                   │
│                    ┌─────────────────┐                          │
│                    │  OTel Collector │  (DaemonSet, 1 per node) │
│                    └────────┬────────┘                          │
└─────────────────────────────┼────────────────────────────────────┘
                              │
                              ▼ Kafka / OTLP
                        Downstream Systems
```

#### How Services Send Data

**Option 1: OpenTelemetry SDK (Recommended)**

Services use the OpenTelemetry SDK which provides a unified API for logs, metrics, and traces:

```java
// Java example
import io.opentelemetry.api.logs.Logger;

Logger logger = GlobalOpenTelemetry.getLogsBridge().loggerBuilder("payment-service").build();

logger.logRecordBuilder()
    .setSeverity(Severity.ERROR)
    .setBody("Payment failed for order " + orderId)
    .setAttribute("order_id", orderId)
    .setAttribute("error_code", "INSUFFICIENT_FUNDS")
    .emit();
```

The SDK:
- Batches log records (doesn't send one at a time)
- Sends via OTLP protocol to localhost (fast, no network hop)
- Handles backpressure gracefully
- Works identically across Java, Python, Go, Node.js, .NET

**Option 2: stdout + Agent Capture**

Services simply write to stdout (12-factor app style):

```python
# Python
print(json.dumps({
    "level": "ERROR",
    "service": "payment-service", 
    "message": "Payment failed",
    "order_id": "12345"
}))
```

The agent tails container logs from `/var/log/containers/*.log` (Kubernetes writes stdout there).

#### Agent Responsibilities

| Responsibility | Description |
|---------------|-------------|
| **Receive** | Accept data via OTLP, file tailing, or other inputs |
| **Parse** | Extract structured fields from log messages |
| **Enrich** | Add metadata: hostname, pod name, namespace, region, environment |
| **Batch** | Combine multiple records into efficient batches |
| **Compress** | Reduce network bandwidth (gzip, snappy) |
| **Buffer** | Store locally on disk if downstream is unavailable |
| **Ship** | Forward to Kafka, Elasticsearch, or other destinations |

#### Agent Technology Choices

| Agent | Best For | Memory Footprint |
|-------|----------|-----------------|
| **OpenTelemetry Collector** | Unified logs + metrics + traces, vendor-neutral | ~50-100 MB |
| **Fluent Bit** | Logs only, extremely lightweight | ~5-10 MB |
| **Vector** | High-throughput, complex transformations | ~50 MB |

#### Why This Pattern Solves Our Problems

| Problem | Solution |
|---------|----------|
| Polyglot complexity | Services use simple OTLP or stdout. Every language works. |
| Kafka client complexity | Agent handles all Kafka logic. Services don't know Kafka exists. |
| Coupling | Services decoupled from backend. Change Kafka to Pulsar? Reconfigure agent only. |
| Reliability | Agent buffers locally during outages. No data loss. |
| Enrichment | Agent adds pod/node metadata automatically. |

---

### 2. Buffering Layer (Kafka)

Kafka acts as the central nervous system, decoupling data producers from consumers and handling traffic bursts.

#### Why Kafka?

| Requirement | How Kafka Helps |
|-------------|-----------------|
| **At-least-once delivery** | Durable, replicated commit log. Data persists until consumers acknowledge. |
| **Handle bursts** | Absorbs Black Friday traffic spikes. Consumers process at their own pace. |
| **Multiple consumers** | Same data feeds Elasticsearch AND Flink alerting via consumer groups. |
| **Replayability** | Debug issues by replaying data from a point in time. |
| **Decoupling** | Producers and consumers evolve independently. |

#### Topic Design

```
Topics:
├── logs              # Application log events
├── metrics           # Numeric time-series data  
├── traces            # Distributed tracing spans
└── alerts            # Generated alert events (from Flink)
```

**Why Separate Topics?**
- Different retention requirements (logs 3 days, metrics 30 days)
- Different consumer processing logic
- Independent scaling of partitions
- Clearer operational boundaries

#### Partition Strategy for `logs` Topic

```
logs topic:
├── partition-0   # service-a, service-d, service-g, ...
├── partition-1   # service-b, service-e, service-h, ...
├── partition-2   # service-c, service-f, service-i, ...
└── ...
```

**Partitioning Key:** Service name

**Why?**
- Logs from same service go to same partition → ordering preserved per service
- Even distribution across partitions (assuming services generate similar volume)
- Flink can do per-service aggregation efficiently (keyed streams)

#### Kafka Sizing

| Parameter | Value | Reasoning |
|-----------|-------|-----------|
| **Partitions (logs)** | 50-100 | Allows parallel consumers, ~50-100 MB/s throughput |
| **Replication Factor** | 3 | Durability across broker failures |
| **Retention** | 24-48 hours | Enough to replay if consumer falls behind |
| **Brokers** | 5-7 | Handle 5 TB/day with headroom for peaks |

#### Consumer Groups

```
Consumer Groups:
├── es-indexer          # Writes to Elasticsearch
├── flink-alerting      # Real-time alerting pipeline  
└── archive-writer      # (Future) Write to S3 for long-term storage
```

Each group reads all data independently. Flink falling behind doesn't affect ES indexing.

---

### 3. Storage Layer (Elasticsearch)

Elasticsearch provides the searchable index that enables developers to query logs in near real-time.

#### Why Elasticsearch?

| Alternative | Why Not |
|-------------|---------|
| **DynamoDB** | Optimized for key-value lookups, not ad-hoc filtered searches. No full-text search. Expensive scans. |
| **PostgreSQL** | Not designed for this scale of writes. Full-text search is an afterthought. |
| **ClickHouse** | Excellent for aggregations, but Elasticsearch is more mature for log search specifically. |
| **Loki** | Lightweight and cheap, but less powerful query capabilities. Good alternative for cost-sensitive deployments. |

**Elasticsearch Strengths:**
- Inverted index enables fast filtered searches
- Horizontal scaling for both reads and writes
- Native time-based index lifecycle management
- Rich query DSL
- Kibana integration for visualization

#### Index Strategy

**Index Naming: Per-Service, Per-Day**

```
Indices:
├── logs-payment-service-2024-12-27
├── logs-payment-service-2024-12-26
├── logs-payment-service-2024-12-25
├── logs-inventory-service-2024-12-27
├── logs-inventory-service-2024-12-26
└── ...
```

**Why Per-Service?**
- Queries almost always filter by service → skip irrelevant indices entirely
- Different services can have different retention policies
- Operational isolation (reindex one service without affecting others)

**Why Per-Day?**
- Easy retention management (delete entire index vs. deleting documents)
- Time-range queries skip old indices automatically
- Shard sizing predictable

#### Index Template Configuration

```json
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0,
      "refresh_interval": "10s",
      "index.lifecycle.name": "logs-policy",
      "index.lifecycle.rollover_alias": "logs-write"
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "service": { "type": "keyword" },
        "level": { "type": "keyword" },
        "message": { "type": "text" },
        "trace_id": { "type": "keyword" },
        "host": { "type": "keyword" },
        "environment": { "type": "keyword" }
      }
    }
  }
}
```

#### Shard Strategy

**Target: 30-50 GB per primary shard**

Why?
- Too small (<10 GB): Overhead of many shards, inefficient queries
- Too large (>50 GB): Slow recovery, uneven cluster distribution
- Sweet spot: Elasticsearch's recommended range

**Rollover-Based Sharding (Recommended)**

Instead of pre-configuring fixed shards, use ILM rollover:

```json
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "40gb",
            "max_age": "1d"
          }
        }
      },
      "delete": {
        "min_age": "3d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

This creates new indices when:
- Current index reaches 40 GB, OR
- Current index is 1 day old

Result: Consistent shard sizes regardless of per-service volume.

#### Cluster Sizing

| Parameter | Value | Calculation |
|-----------|-------|-------------|
| **Daily data** | 5 TB | Given |
| **Retention** | 3 days | 15 TB total |
| **Shard size** | ~40 GB | Best practice |
| **Primary shards** | ~375 | 15 TB ÷ 40 GB |
| **Replicas** | 0 (hot phase) | Prioritize write performance |
| **Total shards** | ~375 | With 0 replicas |
| **Hot data nodes** | 10-12 | ~30-40 shards per node |

**Node Specification:**
- 64 GB RAM (31 GB heap + OS cache)
- 8-16 CPU cores
- NVMe SSDs (high IOPS for indexing)
- 2 TB storage per node

#### Replica Strategy

**0 Replicas for Hot (Write-Heavy) Phase**

Rationale:
- 5 TB/day is write-intensive
- Replicas double indexing cost
- Data is operationally disposable (can re-ingest from Kafka if needed)
- 3-day retention means data is short-lived anyway

**Trade-off Acknowledged:**
If a node dies during Black Friday, you lose several hours of logs. Mitigations:
- Kafka retains data for 24-48 hours → replay and re-index
- Accept this risk for write performance gains

**Alternative: 1 Replica + Relaxed Refresh**
If durability is critical:
```json
{
  "number_of_replicas": 1,
  "refresh_interval": "30s"
}
```
Doubles storage needs but provides HA.

#### Query Performance Optimization

**How a Query Executes:**

```
Query: "ERROR logs from payment-service in last hour"

1. Index Selection
   └── Pattern: logs-payment-service-2024-12-27
       (skips all other services, all other days)

2. Shard Selection  
   └── All shards of selected index (parallelized)

3. Filter Execution
   └── level: "ERROR"
   └── @timestamp: [now-1h TO now]

4. Results Merge
   └── Coordinator combines shard results
```

**Key Optimizations:**
- `service` and `level` are `keyword` fields (exact match, not analyzed)
- Time-based index naming enables index pruning
- Doc values for sorting/aggregations

---

### 4. Alerting System (Apache Flink)

The alerting system detects anomalies in real-time before data even reaches Elasticsearch.

#### Why Stream Processing (Not Polling Elasticsearch)?

| Approach | Latency | Resource Usage | Correctness |
|----------|---------|----------------|-------------|
| **Poll ES every minute** | 60+ seconds | Adds load to ES cluster | Might miss spikes at boundaries |
| **Stream Processing** | Sub-second | Dedicated resources | Accurate windowing |

Stream processing is purpose-built for this problem.

#### Why Apache Flink?

| Framework | Assessment |
|-----------|------------|
| **Apache Flink** | ✅ True streaming, event-time semantics, exactly-once, RocksDB state |
| **Kafka Streams** | Good for simpler cases, but windowing less powerful |
| **Spark Structured Streaming** | Micro-batch = higher latency floor |
| **Custom Consumer** | Maximum control, maximum maintenance burden |

**Flink's Key Advantages:**

1. **Event Time vs Processing Time**
   - "Errors in the last minute" should mean *log timestamp*, not *arrival time*
   - Flink's watermark system handles this correctly

2. **True Streaming**
   - Millisecond-level window evaluation
   - No micro-batch latency floor

3. **Exactly-Once Checkpoints**
   - State survives crashes
   - No duplicate or missed alerts during recovery

4. **State Management**
   - RocksDB-backed state (survives memory pressure)
   - Incremental checkpoints (efficient)

#### Alerting Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           FLINK JOB: Error Rate Alerting                    │
│                                                                             │
│   logs Kafka topic                                                          │
│          │                                                                  │
│          ▼                                                                  │
│   ┌─────────────────┐                                                       │
│   │  Kafka Source   │  (Consumer group: flink-alerting)                     │
│   └────────┬────────┘                                                       │
│            │                                                                │
│            ▼                                                                │
│   ┌─────────────────┐                                                       │
│   │  Filter: 5xx    │  (Only process error logs)                            │
│   │  status codes   │                                                       │
│   └────────┬────────┘                                                       │
│            │                                                                │
│            ▼                                                                │
│   ┌─────────────────┐                                                       │
│   │  Key By:        │  (Partition by service name)                          │
│   │  service_name   │                                                       │
│   └────────┬────────┘                                                       │
│            │                                                                │
│            ▼                                                                │
│   ┌─────────────────────────────────────────────────────────┐              │
│   │  Sliding Window: 1 minute window, 10 second slide       │              │
│   │                                                         │              │
│   │  |----window 1----|                                     │              │
│   │       |----window 2----|                                │              │
│   │            |----window 3----|                           │              │
│   │                                                         │              │
│   │  State (RocksDB):                                       │              │
│   │  ┌─────────────────────────────────────────────────┐   │              │
│   │  │ payment-service:                                 │   │              │
│   │  │   window_counts: [45, 52, 78, 102, 98, 105]    │   │              │
│   │  │   alert_state: FIRING                           │   │              │
│   │  │   last_alert_time: 2024-12-27T10:45:00Z        │   │              │
│   │  │                                                 │   │              │
│   │  │ inventory-service:                              │   │              │
│   │  │   window_counts: [12, 8, 15, 10, 11, 9]        │   │              │
│   │  │   alert_state: OK                               │   │              │
│   │  └─────────────────────────────────────────────────┘   │              │
│   └────────┬────────────────────────────────────────────────┘              │
│            │                                                                │
│            ▼                                                                │
│   ┌─────────────────┐                                                       │
│   │  Threshold      │  (count > 100 in window → trigger)                    │
│   │  Evaluation     │                                                       │
│   └────────┬────────┘                                                       │
│            │                                                                │
│            ▼                                                                │
│   ┌─────────────────┐                                                       │
│   │  Cooldown &     │  (Don't alert again within 10 min)                    │
│   │  Deduplication  │                                                       │
│   └────────┬────────┘                                                       │
│            │                                                                │
│            ▼                                                                │
│   ┌─────────────────┐                                                       │
│   │  Kafka Sink:    │                                                       │
│   │  alerts topic   │                                                       │
│   └─────────────────┘                                                       │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Windowing: Tumbling vs Sliding

**Tumbling Windows (Non-Overlapping)**
```
|----min 1----|----min 2----|----min 3----|
     150           80            200
   → ALERT       → OK         → ALERT
```

Problem: An error spike at minute boundary could be split:
```
|----min 1----|----min 2----|
        ^^^^60 errors^^^^
         (60 each side, neither triggers 100 threshold)
```

**Sliding Windows (Overlapping)**
```
|----window 1----|
     |----window 2----|
          |----window 3----|

Slide: 10 seconds
Window size: 1 minute
```

Advantage: Catches spikes regardless of when they occur.

Trade-off: More state to maintain (6 overlapping windows per minute).

**Our Choice: Sliding Windows**
- Correctness > marginal compute cost
- Per-service state is small (just counters)
- False negatives (missed alerts) are worse than slight CPU overhead

#### Handling Late-Arriving Data

Logs might arrive late due to:
- Network delays
- Agent batching
- Slow producers

**Flink's Watermark System:**

```
Event Time:  10:00:00   10:00:05   10:00:10   10:00:15
                │          │          │          │
                ▼          ▼          ▼          ▼
            log arrives  log arrives  watermark  late log
                                      (10:00:05) (10:00:03)
                                                    │
                                                    ▼
                                           Still processed!
                                           (within allowed lateness)
```

Configuration:
```java
WatermarkStrategy
    .<LogEvent>forBoundedOutOfOrderness(Duration.ofSeconds(30))
    .withTimestampAssigner((event, timestamp) -> event.getTimestamp());
```

Logs up to 30 seconds late are still counted in the correct window.

#### Alert State Machine

```
                    ┌──────────────────────────────────────┐
                    │                                      │
                    ▼                                      │
              ┌──────────┐     threshold      ┌───────────┴┐
              │          │    exceeded        │            │
     ────────▶│    OK    │───────────────────▶│   FIRING   │
              │          │                    │            │
              └──────────┘                    └─────┬──────┘
                    ▲                               │
                    │                               │
                    │      below threshold         │
                    │      for 5 minutes           │
                    └──────────────────────────────┘
```

States:
- **OK:** Normal operation, no alert
- **FIRING:** Threshold exceeded, alert sent
- **COOLDOWN:** (Implicit) Don't re-alert within cooldown period

#### Alert Delivery Architecture

**Why Not Call PagerDuty Directly from Flink?**

| Direct Call | Kafka + Dispatcher |
|-------------|-------------------|
| Side effects in stream processor are risky | Clean separation of concerns |
| If PagerDuty slow, backpressure affects pipeline | Flink stays fast; dispatcher handles retries |
| Hard to add channels (email, SMS) | Just add routing logic |
| No audit trail | `alerts` topic = audit log |

**Alert Dispatcher Service:**

```
┌────────────────────────────────────────────────────────────────┐
│                     ALERT DISPATCHER                           │
│                                                                │
│   alerts Kafka topic                                           │
│          │                                                     │
│          ▼                                                     │
│   ┌─────────────────┐                                          │
│   │  Deduplication  │  Check: Did we already alert for this   │
│   │                 │  service in the last 10 minutes?         │
│   └────────┬────────┘                                          │
│            │                                                   │
│            ▼                                                   │
│   ┌─────────────────┐                                          │
│   │  Routing Rules  │  payment-service → Team A (PagerDuty)   │
│   │                 │  inventory-svc → Team B (Slack)          │
│   └────────┬────────┘                                          │
│            │                                                   │
│            ▼                                                   │
│   ┌─────────────────┐                                          │
│   │  Channel        │                                          │
│   │  Adapters       │                                          │
│   │  ┌───────────┐  │                                          │
│   │  │ PagerDuty │  │                                          │
│   │  ├───────────┤  │                                          │
│   │  │   Slack   │  │                                          │
│   │  ├───────────┤  │                                          │
│   │  │   Email   │  │                                          │
│   │  └───────────┘  │                                          │
│   └─────────────────┘                                          │
└────────────────────────────────────────────────────────────────┘
```

---

### 5. Query Layer

The query layer provides the interface for developers to search and visualize logs.

#### Kibana / OpenSearch Dashboards

Standard UI for:
- Ad-hoc log searches
- Saved queries
- Dashboards with visualizations
- Index pattern management

#### Example Queries

**Find ERROR logs from payment-service:**
```json
GET logs-payment-service-*/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "level": "ERROR" } },
        { "range": { "@timestamp": { "gte": "now-1h" } } }
      ]
    }
  },
  "sort": [{ "@timestamp": "desc" }],
  "size": 100
}
```

**Count errors per service in last hour:**
```json
GET logs-*/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term": { "level": "ERROR" } },
        { "range": { "@timestamp": { "gte": "now-1h" } } }
      ]
    }
  },
  "aggs": {
    "by_service": {
      "terms": { "field": "service", "size": 50 }
    }
  }
}
```

---

## Data Flow Examples

### Example 1: Normal Log Flow

```
1. payment-service logs an error:
   logger.error("Payment failed for order {}", orderId);

2. OTel SDK batches and sends via OTLP to localhost:4317

3. OTel Collector (on same node):
   - Receives log
   - Adds metadata: pod=payment-service-abc123, namespace=prod, region=us-east-1
   - Batches with other logs
   - Compresses and sends to Kafka

4. Kafka:
   - Receives batch
   - Writes to logs topic, partition based on service name
   - Replicates to followers

5. ES Indexer (Kafka consumer):
   - Reads batch from Kafka
   - Transforms to ES bulk format
   - Sends bulk request to Elasticsearch

6. Elasticsearch:
   - Indexes documents into logs-payment-service-2024-12-27
   - Refreshes index (every 10 seconds)
   - Documents now searchable

Total time: 5-10 seconds
```

### Example 2: Alert Triggering

```
1. payment-service experiences elevated errors (>100/min)

2. Flink (parallel consumer):
   - Reads same logs from Kafka
   - Filters to 5xx errors only
   - Keys by service name
   - Sliding window counts: [..., 98, 102, 115, ...]
   - Window count 102 > threshold 100 → TRIGGER

3. Flink checks alert state:
   - payment-service alert_state = OK
   - last_alert_time = null
   - → Transition to FIRING, emit alert event

4. Alert event written to alerts Kafka topic:
   {
     "service": "payment-service",
     "error_count": 102,
     "window_start": "2024-12-27T10:44:00Z",
     "window_end": "2024-12-27T10:45:00Z",
     "severity": "HIGH"
   }

5. Alert Dispatcher:
   - Reads alert from Kafka
   - Checks dedup: No recent alert for payment-service
   - Routes: payment-service → Team A → PagerDuty
   - Calls PagerDuty API

6. On-call engineer receives page

Total time from error spike to page: <30 seconds
```

---

## Scalability & Bottlenecks

### Bottleneck 1: Kafka During Peak Traffic

**Risk:** 3-5x normal traffic during Black Friday could overwhelm brokers.

**Mitigations:**

| Strategy | Description |
|----------|-------------|
| **Pre-scale** | Add partitions and brokers before known peak events |
| **Agent buffering** | OTel Collector buffers locally during short spikes |
| **Dedicated cluster** | Separate Kafka cluster for observability (isolate from business topics) |
| **Sampling** | If desperate, sample DEBUG/INFO logs during extreme peaks |

**Monitoring:**
- Broker CPU/memory utilization
- Under-replicated partitions
- Consumer lag

### Bottleneck 2: Elasticsearch Indexing Lag

**Risk:** At 58 MB/s sustained, ES could fall behind, breaking the 5-10 second SLA.

**Mitigations:**

| Strategy | Description |
|----------|-------------|
| **Bulk size tuning** | 500-1000 documents per bulk request |
| **Refresh interval** | Increase to 10-30 seconds during peaks |
| **Horizontal scaling** | Add data nodes (ES scales linearly for writes) |
| **Thread pool tuning** | Increase bulk thread pool size |

**Monitoring:**
- Indexing rate vs. ingest rate
- Bulk queue rejections
- Refresh latency

### Bottleneck 3: Flink Checkpointing

**Risk:** Large state + frequent checkpoints = slow checkpointing = risk of data loss on failure.

**Mitigations:**

| Strategy | Description |
|----------|-------------|
| **State TTL** | Expire state for inactive services |
| **Incremental checkpoints** | Only checkpoint changes, not full state |
| **Checkpoint interval** | Balance recovery time vs. overhead (1-2 minutes) |
| **RocksDB tuning** | Block cache, write buffer sizes |

**Monitoring:**
- Checkpoint duration
- Checkpoint size
- Backpressure metrics

---

## Trade-offs & Design Decisions

| Decision | Choice | Trade-off |
|----------|--------|-----------|
| **Agent pattern** | OTel Collector | Additional component to manage, but decouples services from infrastructure |
| **Kafka** | Yes | Added complexity, but essential for at-least-once delivery and burst handling |
| **Per-service indices** | Yes | More indices to manage, but query performance and operational isolation |
| **0 Replicas (hot)** | Yes | Risk of data loss, but write performance; mitigated by Kafka replay |
| **Sliding windows** | Yes | More state/compute, but catches edge cases tumbling windows miss |
| **Alerts via Kafka** | Yes | Additional hop, but clean separation and audit trail |

---

## Production Considerations

### Deployment

| Component | Deployment Model |
|-----------|-----------------|
| OTel Collector | Kubernetes DaemonSet (1 per node) |
| Kafka | Dedicated cluster, 5-7 brokers |
| Elasticsearch | Dedicated cluster, 10-12 data nodes |
| Flink | Kubernetes, 3+ TaskManagers |
| Alert Dispatcher | Kubernetes Deployment, 2+ replicas |

### Monitoring the Monitoring System

You need observability for your observability platform:

- **Kafka:** Broker metrics, consumer lag (via Kafka Exporter → Prometheus)
- **Elasticsearch:** Cluster health, indexing rate (via ES Exporter)
- **Flink:** Checkpoint metrics, backpressure (via Flink Metrics)
- **End-to-end:** Synthetic log injection → verify it appears in ES within SLA

### Disaster Recovery

| Scenario | Recovery |
|----------|----------|
| **Kafka broker failure** | Replicas take over, no data loss |
| **ES node failure** | Shard relocation (with replicas) or re-index from Kafka |
| **Flink failure** | Restart from last checkpoint, resume processing |
| **Full AZ outage** | Multi-AZ deployment for all components |

---

## v1 vs v2 Roadmap

### v1: Ship in 4 Weeks

| Include | Exclude |
|---------|---------|
| Logs ingestion only | Metrics & tracing |
| Basic 5xx alerting | Complex anomaly detection |
| Single ES cluster | Hot-warm-cold architecture |
| PagerDuty only | Slack, email channels |
| Fixed thresholds | Adaptive thresholds |
| 3-day retention | Extended retention |
| Basic Kibana | Custom query UI |

**Goal:** Prove pipeline works end-to-end. Get developers using it. Iterate.

### v2: Next Quarter

- Add metrics pipeline (Prometheus/Mimir)
- Add tracing pipeline (Jaeger/Tempo)
- Multi-channel alerting (Slack, email, webhooks)
- Warm/cold tier for extended retention (S3 + restore-on-demand)
- Dynamic thresholds based on historical baselines
- Self-service alert configuration UI

### v3: Future

- ML-based anomaly detection
- Log clustering (group similar errors)
- Correlation across logs/metrics/traces
- Multi-region deployment
- Compliance features (audit logs, access logging, encryption)

---

## Summary

This centralized logging platform handles 5 TB/day of log data with:

1. **Decoupled ingestion** via OpenTelemetry Collectors
2. **Durable buffering** via Kafka
3. **Fast searchable storage** via Elasticsearch with per-service, time-based indices
4. **Real-time alerting** via Apache Flink stream processing
5. **Reliable alert delivery** via dedicated dispatcher service

The design prioritizes:
- **Reliability:** At-least-once delivery, crash recovery
- **Performance:** Sub-10-second searchability, sub-minute alerting
- **Scalability:** Horizontal scaling at every layer
- **Operational simplicity:** Clear component boundaries, standard tooling

