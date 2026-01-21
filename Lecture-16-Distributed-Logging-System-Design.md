# Lecture 16: Distributed Logging System Design

> **Lecture:** 16  
> **Topic:** System Design  
> **Application:** Distributed Logging Platform (Splunk, ELK Stack, OpenObserve)  
> **Scale:** Millions of Events/Hour  
> **Difficulty:** Hard  
> **Previous:** [[Lecture-15-Notification-System-Design|Lecture 15: Notification System]]

---

## 1. Brief Introduction

### What is a Distributed Logging System?

A **distributed logging system** collects, processes, stores, and visualizes logs from multiple microservices across different regions.

**Examples:** Splunk, ELK Stack (Elasticsearch, Logstash, Kibana), OpenObserve, Datadog

### Problem It Solves

1. **Centralized Logs:** All microservice logs in one place
2. **Real-Time Search:** Query logs in < 1 second
3. **Alerting:** Auto-alert on errors/exceptions
4. **Debugging:** Trace requests across services
5. **Compliance:** Retain logs for auditing

---

## 2. Functional and Non-Functional Requirements

### 2.1 Functional Requirements

1. **Multi-Source Ingestion**
   - Ingest logs from multiple microservices globally
   - Support real-time and batch ingestion

2. **Real-Time Ingestion**
   - Logs appear in system within seconds
   - Agent-based continuous streaming

3. **Batch Ingestion (Offline)**
   - For offline devices/legacy systems
   - Upload log files periodically

4. **Validation & Normalization**
   - Validate log format
   - Parse and standardize logs
   - Enrich with metadata

5. **Search Dashboard**
   - Search by keyword, service, time range
   - Filter by log level (INFO, ERROR, WARN)

### 2.2 Out of Scope

- Log analytics/ML (anomaly detection)
- Distributed tracing (use Jaeger/Zipkin)

### 2.3 Non-Functional Requirements

1. **Scale**
   - **Millions of events/hour**
   - Support 1000s of microservices

2. **Low Latency**
   - Log ingestion: < 1 second
   - Search query: < 2 seconds

3. **CAP Theorem**
   - **Highly Available (AP):** Eventual consistency acceptable
   
   **Why?**
   - ✅ Logs can arrive with 1-2 sec delay
   - ✅ Downtime = Cannot ingest logs ❌

4. **Durability (with TTL)**
   - No data loss during ingestion
   - Retention: 45 days (configurable)

---

## 3. Core Entities

1. **Event** - Log entry
2. **Ingestion Job** - Batch upload job
3. **Agent** - Log collector (FluentBit, Filebeat)
4. **Client/Organization** - Registered tenant (Amazon, Uber)

---

## 4. Log Flow (How Logs Reach Backend)

### 4.1 Log Journey

```
┌─────────────────────────────────────────────────────────┐
│         MICROSERVICE (Java, Node.js, Python)             │
│              log4j.info("Order created")                 │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │   stdout/stderr │
                │    (IO Stream)  │
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │   Kubernetes    │
                │   (kubectl logs)│
                │ Writes to file  │
                └────────┬────────┘
                         │
                         ▼
                 /var/log/pods/
                   raw.log
                         │
                         ▼
                ┌─────────────────┐
                │  Log Agent      │
                │  (FluentBit,    │
                │   Filebeat)     │
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │  Logging Backend│
                │   (Splunk, ELK) │
                └─────────────────┘
```

---

### 4.2 Detailed Flow

```
Step 1: Application logs
    log4j.info("User login successful")
    ↓
Step 2: Writes to stdout
    System.out.println(...)
    ↓
Step 3: Kubernetes captures stdout
    kubectl logs pod-xyz
    Stored in: /var/log/pods/<namespace>/<pod>/container.log
    ↓
Step 4: Agent reads log file
    FluentBit tails /var/log/pods/**/*.log
    ↓
Step 5: Agent sends to backend
    HTTP/gRPC to logging service
```

---

## 5. API Design

### 5.1 Agent Onboarding

#### **Register Agent**

```
POST /v1/agents/register
```

**Request Body:**
```json
{
  "organizationName": "Amazon",
  "environment": "PRODUCTION",  // DEV, STAGING, PROD
  "agentVersion": "1.2.0"
}
```

**Response:**
```json
{
  "clientId": "client_abc123",
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "tokenTTL": 86400  // 24 hours
}
```

---

#### **Get Agent Configuration**

```
GET /v1/agents/{agentId}/config
```

---

#### **Heartbeat**

```
POST /v1/agents/{agentId}/heartbeat
```

---

### 5.2 Log Ingestion

#### **Real-Time Ingestion (Agent)**

```
POST /v1/logs/ingest
```

**Request Body:**
```json
{
  "clientId": "client_abc123",
  "token": "...",
  "logs": [
    {
      "timestamp": "2026-01-21T10:30:45Z",
      "level": "ERROR",
      "service": "checkout-service",
      "message": "Payment gateway timeout",
      "traceId": "trace_xyz789",
      "metadata": {
        "userId": "user_123",
        "orderId": "order_456"
      }
    }
  ]
}
```

---

#### **Batch Ingestion (File Upload)**

```
POST /v1/logs/upload
```

**Request:**
```
Content-Type: multipart/form-data

file: logs_2026-01-20.txt.gz
```

---

### 5.3 Search APIs

#### **Search Logs**

```
GET /v1/logs/search?query={query}&from={timestamp}&to={timestamp}&level={level}&service={service}&page={page}
```

**Example:**
```
GET /v1/logs/search?query=timeout&from=2026-01-21T00:00:00Z&to=2026-01-21T23:59:59Z&level=ERROR&service=checkout
```

**Response:**
```json
{
  "logs": [
    {
      "timestamp": "2026-01-21T10:30:45Z",
      "level": "ERROR",
      "service": "checkout-service",
      "message": "Payment gateway timeout",
      "traceId": "trace_xyz789"
    }
  ],
  "total": 1250,
  "page": 1
}
```

---

#### **Tail Logs (Real-Time - REST)**

```
GET /v1/logs/tail?service={service}&level={level}
```

**Poll every 5 seconds from frontend**

---

#### **Tail Logs (Real-Time - WebSocket)**

```
WS /v1/logs/tail
```

**Message:**
```json
{
  "subscribe": {
    "service": "checkout-service",
    "level": "ERROR"
  }
}
```

**Server pushes logs in real-time**

---

## 6. High-Level Design (HLD)

### 6.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│              MICROSERVICES (Global)                      │
│         (checkout, payment, notification)                │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │   Log Agents    │
                │   (FluentBit)   │
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │   API Gateway   │
                │  Load Balancer  │
                └────────┬────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ Client   │    │Ingestion │    │  Search  │
  │Onboarding│    │ Service  │    │ Service  │
  └────┬─────┘    └────┬─────┘    └────┬─────┘
       │               │               │
       ▼               ▼               ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │Client DB │    │  Kafka   │    │Elastic-  │
  └──────────┘    │          │    │ search   │
                  └────┬─────┘    └──────────┘
                       │
                       ▼
                  ┌──────────┐
                  │ Log DB   │
                  │(Cassandra│
                  └──────────┘
```

---

## 7. Low-Level Design (LLD)

### 7.1 Client Onboarding Service

**Purpose:** Register organizations, issue tokens

**Client DB (PostgreSQL):**

```sql
CREATE TABLE clients (
    client_id UUID PRIMARY KEY,
    organization_name VARCHAR(100),
    token VARCHAR(500),  -- JWT
    token_ttl BIGINT,  -- Expiry timestamp
    environment VARCHAR(20),  -- DEV, STAGING, PROD
    alert_preferences JSONB,  -- {"pagerduty": true, "email": true}
    created_at TIMESTAMP
);

CREATE INDEX idx_token ON clients(token);
```

---

**Flow:**

```
1. Organization registers (POST /v1/agents/register)
    ↓
2. Client Onboarding Service:
    - Generate client_id (UUID)
    - Generate JWT token (24h TTL)
    - Store in Client DB
    ↓
3. Return client_id + token
    ↓
4. Organization configures FluentBit:
    [OUTPUT]
        Name http
        Match *
        Host logging.example.com
        Port 443
        URI /v1/logs/ingest
        Header Authorization Bearer <token>
```

---

### 7.2 Ingestion Service (Two Types)

#### **7.2.1 Agent-Based Service (Real-Time)**

**Purpose:** Accept logs from agents (FluentBit, Filebeat)

**Flow:**

```
┌─────────────────────────────────────────────────────────┐
│              FLUENTBIT AGENT                             │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │ Agent-Based     │
                │   Service       │
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │     Kafka       │
                │  (raw_logs)     │
                └─────────────────┘
```

---

#### **7.2.2 File Injection Service (Batch/Offline)**

**Purpose:** Upload log files from offline systems

**Flow:**

```
Offline device collects logs locally
    ↓
Device comes online
    ↓
POST /v1/logs/upload (file: logs.txt.gz)
    ↓
File Injection Service → Kafka (raw_logs)
```

---

### 7.3 Kafka (Log Buffer)

**Topics:**
- `raw_logs` - Unprocessed logs from agents
- `alert_logs` - Error logs that need alerting

**Why Kafka?**
- ✅ Buffer millions of logs/hour
- ✅ Decouple ingestion from processing
- ✅ Replay logs if processing fails

---

### 7.4 Apache Flink (Stream Processor) ⭐

**Purpose:** Validate, parse, normalize, enrich logs

**Responsibilities:**

1. **Validation**
   - Check required fields (timestamp, level, service)
   - Discard malformed logs

2. **Parsing**
   - Extract structured data from unstructured logs
   - Example: `"User 123 logged in"` → `{userId: 123, action: "login"}`

3. **Normalization**
   - Convert to standard format
   - Example: `2026/01/21` → `2026-01-21T00:00:00Z` (ISO 8601)

4. **Enrichment**
   - Add metadata (region, cluster, pod name)

5. **Deduplication**
   - Remove duplicate logs (same timestamp + message)

---

**Flink Flow:**

```
┌─────────────────────────────────────────────────────────┐
│                    KAFKA (raw_logs)                      │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │  Apache Flink   │
                │                 │
                │  1. Validate    │
                │  2. Parse       │
                │  3. Normalize   │
                │  4. Enrich      │
                │  5. Deduplicate │
                └────────┬────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │Elastic-  │    │ Log DB   │    │    S3    │
  │ search   │    │(Cassandra│    │  Bucket  │
  │(14 days) │    │(45 days) │    │(Long-term│
  └──────────┘    └──────────┘    └──────────┘
```

---

**Flink Pseudo-Code:**

```java
DataStream<RawLog> rawLogs = kafka.consume("raw_logs");

DataStream<ValidLog> validatedLogs = rawLogs
    .filter(log -> log.hasRequiredFields())  // Validation
    .map(log -> parseLog(log))               // Parsing
    .map(log -> normalizeLog(log))           // Normalization
    .map(log -> enrichLog(log))              // Enrichment
    .keyBy(log -> log.getMessageHash())
    .window(TumblingEventTimeWindows.of(Time.seconds(5)))
    .reduce((log1, log2) -> log1)           // Deduplication

// Write to 3 sinks
validatedLogs.addSink(elasticsearchSink);   // Recent logs
validatedLogs.addSink(cassandraSink);       // Medium-term
validatedLogs.addSink(s3Sink);              // Long-term
```

---

### 7.5 Three-Tier Storage Strategy ⭐

**Key Concept:** Hot, Warm, Cold storage based on data age

| Storage | Data Age | Use Case | Cost | Query Speed |
|---------|----------|----------|------|-------------|
| **Elasticsearch** | < 14 days | Real-time search | High | < 1 sec |
| **Cassandra** | 15-45 days | Recent history | Medium | < 5 sec |
| **S3** | > 45 days | Compliance/Audit | Low | > 10 sec |

---

#### **7.5.1 Elasticsearch (Hot Storage)**

**Purpose:** Fast search for recent logs (< 14 days)

**Why Elasticsearch?**
- ✅ Full-text search (< 1 second)
- ✅ Inverted index for keywords
- ✅ Built-in TTL (auto-delete old docs)

**Index Structure:**

```json
PUT /logs-2026-01-21
{
  "mappings": {
    "properties": {
      "timestamp": {"type": "date"},
      "level": {"type": "keyword"},
      "service": {"type": "keyword"},
      "message": {"type": "text"},
      "traceId": {"type": "keyword"},
      "metadata": {"type": "object"}
    }
  }
}
```

**TTL Strategy:**
```
Index per day: logs-2026-01-21, logs-2026-01-22, ...
Keep only last 14 indices
Delete indices older than 14 days (cron job)
```

---

#### **7.5.2 Cassandra (Warm Storage)**

**Purpose:** Medium-term storage (15-45 days)

**Why Cassandra?**
- ✅ Write-optimized (millions of writes/hour)
- ✅ Time-series data
- ✅ Horizontal scaling

**Schema:**

```sql
CREATE TABLE logs_by_service (
    service VARCHAR,
    timestamp TIMESTAMP,
    log_id UUID,
    level VARCHAR,
    message TEXT,
    trace_id VARCHAR,
    metadata MAP<TEXT, TEXT>,
    PRIMARY KEY ((service), timestamp, log_id)
) WITH CLUSTERING ORDER BY (timestamp DESC);

CREATE INDEX ON logs_by_service(level);
CREATE INDEX ON logs_by_service(trace_id);
```

**Partitioning:**
- Partition key: `service` (distribute by microservice)
- Clustering key: `timestamp` (sort by time)

**Query Example:**
```sql
SELECT * FROM logs_by_service
WHERE service = 'checkout-service'
AND timestamp >= '2026-01-10T00:00:00Z'
AND timestamp <= '2026-01-15T23:59:59Z'
AND level = 'ERROR'
LIMIT 100;
```

---

#### **7.5.3 S3 (Cold Storage)**

**Purpose:** Long-term archival (> 45 days)

**Why S3?**
- ✅ Cheapest storage ($0.023/GB/month)
- ✅ Durable (99.999999999%)
- ✅ Compliance (GDPR, SOC2)

**Structure:**

```
s3://logs-archive/
├── year=2026/
│   ├── month=01/
│   │   ├── day=21/
│   │   │   ├── service=checkout/
│   │   │   │   ├── logs-00001.json.gz
│   │   │   │   ├── logs-00002.json.gz
```

**Querying S3:**
- Use AWS Athena (SQL over S3)
- Slower (10+ seconds), but acceptable for old logs

---

### 7.6 Data Lifecycle

```
Day 0-14: Elasticsearch (fast search)
    ↓
Day 15-45: Cassandra (medium speed)
    ↓
Day 46+: S3 (slow, cheap)
    ↓
Day 365+: Delete (or move to Glacier)
```

**Automatic Cleanup:**

```
Cron Job (runs daily at 2 AM):
1. Delete ES indices older than 14 days
2. Delete Cassandra rows older than 45 days
3. Optionally: Move S3 to Glacier after 1 year
```

---

### 7.7 Search Service

**Architecture:**

```
┌─────────────────────────────────────────────────────────┐
│                  USER DASHBOARD                          │
│            Search: "timeout" [ERROR] [checkout]          │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │ Search Service  │
                └────────┬────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │Elastic-  │    │Cassandra │    │    S3    │
  │ search   │    │          │    │ (Athena) │
  │(< 14d)   │    │(15-45d)  │    │  (>45d)  │
  └──────────┘    └──────────┘    └──────────┘
```

---

**Query Flow:**

```java
List<Log> searchLogs(String query, String service, String level, long fromTs, long toTs) {
    long now = System.currentTimeMillis();
    long fourteenDaysAgo = now - (14 * 24 * 3600 * 1000);
    long fortyFiveDaysAgo = now - (45 * 24 * 3600 * 1000);
    
    List<Log> results = new ArrayList<>();
    
    // Hot data: Query Elasticsearch
    if (toTs >= fourteenDaysAgo) {
        results.addAll(elasticsearchClient.search(query, service, level, fromTs, toTs));
    }
    
    // Warm data: Query Cassandra
    if (fromTs < fourteenDaysAgo && toTs >= fortyFiveDaysAgo) {
        results.addAll(cassandraClient.query(service, level, fromTs, toTs));
    }
    
    // Cold data: Query S3 (Athena)
    if (fromTs < fortyFiveDaysAgo) {
        results.addAll(athenaClient.query(service, level, fromTs, toTs));
    }
    
    return results;
}
```

---

### 7.8 Alerting System

**Purpose:** Auto-alert on error logs

**Architecture:**

```
┌─────────────────────────────────────────────────────────┐
│                  APACHE FLINK                            │
│         (Detects ERROR logs)                             │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │     Kafka       │
                │  (alert_logs)   │
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │ Alert Service   │
                └────────┬────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ Alert    │    │ Client   │    │Notification
  │ Rules DB │    │   DB     │    │  Service │
  └──────────┘    └──────────┘    └────┬─────┘
                                       │
                                       ▼
                               ┌──────────────┐
                               │Email/Pager/  │
                               │Slack/PagerDuty
                               └──────────────┘
```

---

**Alert Rules DB (PostgreSQL):**

```sql
CREATE TABLE alert_rules (
    rule_id UUID PRIMARY KEY,
    client_id UUID,
    service VARCHAR(100),
    condition JSONB,  -- {"level": "ERROR", "count": ">10", "window": "5min"}
    severity VARCHAR(20),  -- P0, P1, P2
    notification_channels JSONB,  -- ["email", "pagerduty"]
    created_at TIMESTAMP
);
```

**Example Rule:**
```json
{
  "service": "checkout-service",
  "condition": {
    "level": "ERROR",
    "count": ">10",
    "window": "5min"
  },
  "severity": "P0",
  "channels": ["pagerduty", "slack"]
}
```

---

**Alert Flow:**

```
Flink detects ERROR log
    ↓
Publish to Kafka (alert_logs)
    ↓
Alert Service consumes
    ↓
Check Alert Rules DB:
    - Is this error critical?
    - Has threshold been crossed? (e.g., >10 errors in 5 mins)
    ↓
If YES:
    - Fetch client preferences (Client DB)
    - Send to Notification Service
    ↓
Notification Service:
    - Send email via SendGrid
    - Create PagerDuty incident
    - Post to Slack channel
```

---

## 8. Complete Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│          MICROSERVICES (Global, 1000s)                   │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │  FluentBit      │
                │   Agents        │
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │   API Gateway   │
                └────────┬────────┘
                         │
  ┌──────────────────────┼──────────────────────┐
  │                      │                      │
  ▼                      ▼                      ▼
┌──────────┐      ┌──────────┐          ┌──────────┐
│ Client   │      │Agent-Based│          │  File    │
│Onboarding│      │ Service  │          │Injection │
└────┬─────┘      └────┬─────┘          └────┬─────┘
     │                 │                     │
     ▼                 └──────────┬──────────┘
┌──────────┐                      │
│Client DB │                      ▼
└──────────┘              ┌──────────────┐
                         │    Kafka     │
                         │  (raw_logs)  │
                         └────┬─────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │  Apache Flink    │
                    │ (Validate/Parse/ │
                    │  Normalize)      │
                    └────┬─────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │Elastic-  │    │Cassandra │    │    S3    │
  │search    │    │  (45d)   │    │(Archive) │
  │ (14d)    │    └──────────┘    └──────────┘
  └────┬─────┘
       │
       ▼
  ┌──────────┐
  │ Search   │
  │ Service  │
  └──────────┘


  ┌──────────────────────────────────────┐
  │         ALERTING SYSTEM               │
  │                                       │
  │  Flink → Kafka (alert_logs)          │
  │      ↓                                │
  │  Alert Service                        │
  │      ↓                                │
  │  Notification Service → PagerDuty    │
  └──────────────────────────────────────┘
```

---

## 9. Interview Q&A

### Q1: Why three storage layers? Why not just Elasticsearch?

**A:**

**Cost Comparison (1TB/month):**

| Storage | Cost | Total for 1 year |
|---------|------|------------------|
| Elasticsearch | $100/TB | $1,200 |
| Cassandra | $30/TB | $360 |
| S3 | $23/TB | $276 |

**For 1TB logs/day:**
```
Elasticsearch only: $100 × 365 = $36,500/year ❌

Three-tier:
- ES (14 days): $100 × 14 = $1,400
- Cassandra (31 days): $30 × 31 = $930
- S3 (320 days): $23 × 320 = $7,360
Total: $9,690/year ✅ (73% savings!)
```

### Q2: Why Apache Flink instead of custom consumers?

**A:**

**Flink Advantages:**
- ✅ Built-in state management
- ✅ Exactly-once processing
- ✅ Windowing for deduplication
- ✅ Backpressure handling
- ✅ Horizontal scaling

**Alternative:** Write 5 separate consumers (validation, parsing, normalization, enrichment, deduplication) = Complex!

### Q3: How to handle millions of logs/hour?

**A:**

**Scaling Strategies:**

1. **Kafka Partitioning:**
   ```
   raw_logs topic: 100 partitions
   Each partition: 10,000 logs/hour
   ```

2. **Flink Parallelism:**
   ```
   100 Flink task slots
   Each processes 10,000 logs/hour
   ```

3. **Elasticsearch Sharding:**
   ```
   Index per day: logs-2026-01-21
   5 shards per index
   ```

4. **Cassandra Partitioning:**
   ```
   Partition by service name
   checkout-service, payment-service, etc.
   ```

### Q4: What if Flink crashes during processing?

**A:**

**Kafka to the rescue!**

```
Kafka stores logs for 7 days
    ↓
Flink crashes
    ↓
Flink restarts
    ↓
Resumes from last committed offset
    ↓
No data loss ✅
```

### Q5: How to search across all three storage layers?

**A:**

**Search Service logic:**

```java
if (timeRange overlaps with last 14 days) {
    results += searchElasticsearch(...);
}

if (timeRange overlaps with 15-45 days) {
    results += searchCassandra(...);
}

if (timeRange overlaps with 45+ days) {
    results += searchS3Athena(...);
}

return results;
```

### Q6: How to prevent duplicate logs?

**A:**

**Flink Deduplication:**

```java
// Hash based on timestamp + message + service
String hash = md5(log.getTimestamp() + log.getMessage() + log.getService());

// 5-second tumbling window
.keyBy(log -> hash)
.window(TumblingEventTimeWindows.of(Time.seconds(5)))
.reduce((log1, log2) -> log1)  // Keep first, discard duplicates
```

### Q7: How to handle log spikes (e.g., error storm)?

**A:**

**Kafka acts as buffer:**

```
Normal: 10,000 logs/sec → Flink processes at 10,000/sec
    ↓
Spike: 100,000 logs/sec → Kafka buffers
    ↓
Flink processes at 10,000/sec (backpressure)
    ↓
Queue clears in ~10 minutes
```

**Alternative:** Auto-scale Flink based on Kafka lag

### Q8: WebSocket vs REST polling for real-time logs?

**A:**

| Method | Pros | Cons |
|--------|------|------|
| **REST Polling** | Simple, stateless | Wasteful (empty responses), latency |
| **WebSocket** | Real-time push, efficient | Stateful, complex |

**Recommendation:** WebSocket for real-time, REST for dashboard

### Q9: How to ensure log ordering?

**A:**

**Kafka guarantees order within partition:**

```
Partition by service name
    ↓
All logs from "checkout-service" → Partition 5
    ↓
Consumer reads in order ✅
```

**Timestamp-based ordering in Elasticsearch/Cassandra:**
```
ORDER BY timestamp DESC
```

### Q10: Security considerations?

**A:**

1. **Authentication:** JWT tokens (24h TTL)
2. **Authorization:** Client can only see own logs
3. **Encryption:** TLS in transit, AES-256 at rest
4. **PII Masking:** Redact sensitive fields (SSN, credit card)
5. **Audit Logs:** Log all search queries

---

## 10. Key Takeaways

✅ **Log Flow:** App → stdout → Kubernetes → Agent → Backend  
✅ **Kafka Buffer:** Decouple ingestion from processing  
✅ **Apache Flink:** Validate, parse, normalize, enrich, deduplicate  
✅ **Three-Tier Storage:** ES (14d) + Cassandra (45d) + S3 (archive)  
✅ **Cost Optimization:** 73% savings with tiered storage  
✅ **Real-Time Search:** Elasticsearch for < 14 days  
✅ **Alerting:** Flink → Kafka (alert_logs) → PagerDuty  
✅ **Scalability:** Kafka partitions + Flink parallelism  
✅ **Durability:** Kafka retains logs for replay  
✅ **WebSocket:** Real-time log streaming to dashboard  

---

## Summary

**Architecture Highlights:**
- 3 microservices (Client Onboarding, Ingestion, Search)
- 3 storage layers (Elasticsearch, Cassandra, S3)
- Kafka for buffering
- Apache Flink for stream processing
- Alerting system with PagerDuty integration

**Log Journey:**
```
Microservice → Agent (FluentBit) → Kafka
    ↓
Apache Flink (validate + parse + normalize)
    ↓
Elasticsearch (14d) + Cassandra (45d) + S3 (archive)
    ↓
Search Service → Dashboard
```

**Storage Strategy:**
```
Hot (0-14d): Elasticsearch (fast, expensive)
Warm (15-45d): Cassandra (medium, medium cost)
Cold (45d+): S3 (slow, cheap)
```

**Alerting Flow:**
```
ERROR log → Flink → Kafka (alert_logs)
    ↓
Alert Service checks rules
    ↓
Notification Service → PagerDuty/Slack/Email
```

**Performance:**
- Millions of logs/hour
- Search latency: < 2 seconds
- Ingestion latency: < 1 second

**End of Lecture 16**
