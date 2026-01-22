# Lecture 9: Distributed Job Scheduler System Design

> **Lecture:** 09  
> **Topic:** System Design  
> **Application:** Distributed Job Scheduler (Cron, Airflow, Jenkins)  
> **Scale:** 10,000 Jobs/Second  
> **Difficulty:** Hard  
> **Previous:** [[Lecture-08-Chat-Application-System-Design|Lecture 8: Chat Application]]

---

## 📋 Table of Contents

1. [Brief Introduction](#1-brief-introduction)
2. [Functional and Non-Functional Requirements](#2-functional-and-non-functional-requirements)
3. [Core Entities](#3-core-entities)
4. [API Design](#4-api-design)
5. [High-Level Design (HLD)](#5-high-level-design-hld)
6. [Low-Level Design (LLD)](#6-low-level-design-lld)
7. [Complete Architecture Diagram](#7-complete-architecture-diagram)
8. [Database Indexing & Partitioning](#8-database-indexing--partitioning)
9. [Interview Q&A](#9-interview-qa)
10. [Key Takeaways](#10-key-takeaways)

---

<br>

## 1. Brief Introduction

### What is a Distributed Job Scheduler?

A **distributed job scheduler** is a system that allows users to create, schedule, and execute repetitive or one-time jobs at specific times across multiple servers.

**Examples:** Cron, Apache Airflow, Jenkins, Kubernetes CronJobs, AWS Batch

### Problem It Solves

1. **Scheduled Execution:** Run jobs at specific times
2. **Recurring Jobs:** Daily, weekly, monthly tasks
3. **Immediate Execution:** Run jobs on-demand
4. **Monitoring:** Track job status in real-time
5. **Retry Logic:** Handle failures gracefully
6. **Scalability:** Handle thousands of concurrent jobs

**Use Cases:**
- **Data Pipelines:** ETL jobs every night at 2 AM
- **Database Cleanup:** Delete old records every Sunday
- **Report Generation:** Monthly sales reports
- **Backup Jobs:** Daily database backups
- **Email Campaigns:** Send newsletters every Friday

---

## 2. Functional and Non-Functional Requirements

### 2.1 Functional Requirements

1. **Create and Schedule Jobs**
   - Run immediately
   - Run at future time
   - Run on schedule (cron expression)

2. **Monitor Jobs**
   - Real-time status updates
   - View job history
   - See error logs

3. **Cancel and Update Jobs**
   - Cancel scheduled jobs
   - Cancel running jobs
   - Update job schedule

⚠️ **Out of Scope:** Job dependencies (DAGs like Airflow)

### 2.2 Non-Functional Requirements

1. **Scale**
   - **10,000 jobs/second** (peak traffic)
   - Support concurrent execution

2. **CAP Theorem**
   - **Highly Available (AP):** Zero downtime
   - **Eventual Consistency:** Slight delay in job listing is acceptable
   - **Trade-off:** Availability > Consistency

   **Why?**
   - ✅ Job scheduler must always be available
   - ✅ Eventual consistency is acceptable (job appears in dashboard after 1-2 seconds)

3. **At-Least-Once Execution**
   - **Every scheduled job must run at least once**
   - No job should be skipped

4. **Low Latency**
   - **Execute jobs within 2 seconds of scheduled time**
   - Example: Job scheduled at 5:30:00 → Must start by 5:30:02

---

## 3. Core Entities

1. **Job** - Task to be executed
2. **Scheduler** - Determines when to run jobs
3. **Executor** - Runs the actual job

---

## 4. API Design

### 4.1 Create Job

```
POST /v1/jobs
```

**Request Body:**
```json
{
  "name": "Daily Cleanup",
  "scheduleType": "cron",  // immediate, future, cron
  "scheduleTime": "2026-01-22T02:00:00Z",  // For future jobs
  "cronExpression": "0 0 2 * * *",  // For cron jobs
  "payload": {
    "script": "cleanup.py",
    "params": {"days": 30}
  },
  "maxRetries": 3
}
```

**Response:**
```json
{
  "jobId": "job_123",
  "status": "scheduled"
}
```

---

### 4.2 Get Job Details

```
GET /v1/jobs/{jobId}
```

**Response:**
```json
{
  "jobId": "job_123",
  "name": "Daily Cleanup",
  "scheduleType": "cron",
  "cronExpression": "0 0 2 * * *",
  "status": "scheduled",
  "createdAt": "2026-01-21T12:00:00Z"
}
```

---

### 4.3 Get Job Status

```
GET /v1/jobs/{jobId}/status
```

**Response:**
```json
{
  "jobId": "job_123",
  "status": "running",  // queued, running, success, failed
  "startTime": "2026-01-22T02:00:01Z",
  "progress": "50%",
  "executorId": "executor_456"
}
```

---

### 4.4 Update Job

```
PUT /v1/jobs/{jobId}
```

**Request Body:**
```json
{
  "cronExpression": "0 0 3 * * *",  // Change to 3 AM
  "maxRetries": 5
}
```

---

### 4.5 Cancel Job

```
DELETE /v1/jobs/{jobId}
```

**Response:**
```json
{
  "jobId": "job_123",
  "status": "cancelled"
}
```

⚠️ **Note:** Can cancel both scheduled and running jobs

---

### 4.6 Run Job Immediately

```
POST /v1/jobs/{jobId}/run
```

**Response:**
```json
{
  "jobId": "job_123",
  "runId": "run_789",
  "status": "queued"
}
```

---

## 5. High-Level Design (HLD)

### 5.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Users                                 │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │  API Gateway    │
                │  Load Balancer  │
                └────────┬────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │   Job    │    │  Search  │    │ Watcher  │
  │ Service  │    │ Service  │    │ Service  │
  └────┬─────┘    └────┬─────┘    └────┬─────┘
       │               │                │
       ▼               ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │  Kafka   │    │  Job DB  │    │  Kafka   │
  │          │    │(Postgres)│    │  (Queue) │
  └────┬─────┘    └──────────┘    └────┬─────┘
       │                                │
       ▼                                ▼
  ┌──────────┐                    ┌──────────┐
  │ Consumer │                    │   Job    │
  │ Service  │                    │ Consumer │
  └────┬─────┘                    └────┬─────┘
       │                                │
       ▼                                ▼
  ┌──────────┐                    ┌──────────┐
  │  Job DB  │                    │ Executor │
  │(Postgres)│                    │ Service  │
  └──────────┘                    └──────────┘
```

### 5.2 Service Responsibilities

**Job Service:**
- Create, update, delete jobs
- Trigger immediate execution

**Search Service:**
- Get job details
- Get job status

**Watcher Service:**
- Scan Job DB every 20 seconds
- Fetch jobs to run in next 5 minutes
- Push to Kafka queue

**Job Consumer:**
- Consume jobs from Kafka
- Send to Executor Service

**Executor Service:**
- Execute actual job
- Update status every 10 seconds

---

## 6. Low-Level Design (LLD)

### 6.1 Job Service (Create/Update/Delete)

**Flow:**

```
User creates job
    ↓
POST /v1/jobs
    ↓
Job Service
    ↓
Publish to Kafka (job_events topic)
    ↓
Consumer Service
    ↓
Insert into Job DB
```

**Why Kafka?**
- ✅ Handle 10,000 jobs/second
- ✅ Decouple write operations
- ✅ Prevent database overload

---

### 6.2 Job Database (PostgreSQL)

**Why PostgreSQL?**
- ✅ Relational data (jobs, runs, schedules)
- ✅ ACID compliance
- ✅ Indexing support

#### **Table 1: Jobs**

```sql
CREATE TABLE jobs (
    job_id UUID PRIMARY KEY,
    name VARCHAR(255),
    schedule_type VARCHAR(20),  -- immediate, future, cron
    schedule_time TIMESTAMP,  -- For future jobs
    cron_expression VARCHAR(100),  -- For cron jobs
    status VARCHAR(20),  -- scheduled, paused, cancelled
    payload JSONB,  -- Job parameters
    max_retries INT DEFAULT 3,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

-- Index for Watcher Service
CREATE INDEX idx_schedule_time ON jobs(schedule_time);
CREATE INDEX idx_job_id ON jobs(job_id);
```

**Cron Expression Format:**

```
* * * * * *
│ │ │ │ │ │
│ │ │ │ │ └─ Day of week (0-6, Sunday=0)
│ │ │ │ └─── Month (1-12)
│ │ │ └───── Day of month (1-31)
│ │ └─────── Hour (0-23)
│ └───────── Minute (0-59)
└─────────── Second (0-59)
```

**Examples:**
```
0 0 2 * * *   → Every day at 2:00 AM
0 30 9 * * 1  → Every Monday at 9:30 AM
0 0 0 1 * *   → First day of every month at midnight
```

---

#### **Table 2: Job Runs**

```sql
CREATE TABLE job_runs (
    id SERIAL PRIMARY KEY,  -- Auto-increment
    job_id UUID REFERENCES jobs(job_id),
    status VARCHAR(20),  -- queued, running, success, failed
    start_time TIMESTAMP,
    end_time TIMESTAMP,
    modified_time TIMESTAMP,  -- Last heartbeat
    executor_id VARCHAR(100),
    attempt_number INT,
    error_message TEXT,
    created_at TIMESTAMP
);

-- Composite index for Watcher Service
CREATE INDEX idx_status_modified ON job_runs(status, modified_time);
CREATE INDEX idx_job_id ON job_runs(job_id);
```

**Why `modified_time`?**
- Track last heartbeat from Executor
- Detect dead executors (no update for 15+ seconds)

---

### 6.3 Watcher Service (The Heart of the System!)

**Responsibilities:**

1. **Fetch upcoming jobs** (every 20 seconds)
2. **Detect dead executors** (check `modified_time`)

---

#### **Task 1: Fetch Upcoming Jobs**

**Algorithm:**

```python
def fetch_upcoming_jobs():
    # Run every 20 seconds
    while True:
        current_time = now()
        future_time = current_time + 5 minutes
        
        # Fetch jobs to run in next 5 minutes
        jobs = db.query("""
            SELECT * FROM jobs
            WHERE schedule_time BETWEEN ? AND ?
            AND status = 'scheduled'
        """, current_time, future_time)
        
        # Push to Kafka
        for job in jobs:
            kafka.publish("run_queue", job)
            
            # Insert into job_runs table
            db.execute("""
                INSERT INTO job_runs (job_id, status, created_at)
                VALUES (?, 'queued', NOW())
            """, job.job_id)
        
        sleep(20 seconds)
```

**Why 5-minute window?**
- Balance between database load and latency
- Ensures jobs are picked up in time

**Why 20-second interval?**
- Prevents database overload
- Acceptable latency for most use cases

---

#### **Task 2: Detect Dead Executors**

**Problem:** Executor crashes while running job

**Solution:** Check `modified_time`

**Algorithm:**

```python
def detect_dead_executors():
    # Run every 20 seconds
    while True:
        threshold = now() - 15 seconds
        
        # Find jobs with stale heartbeat
        stale_jobs = db.query("""
            SELECT * FROM job_runs
            WHERE status = 'running'
            AND modified_time < ?
        """, threshold)
        
        # Re-queue for retry
        for job in stale_jobs:
            kafka.publish("retry_queue", job)
            
            # Increment attempt number
            db.execute("""
                UPDATE job_runs
                SET attempt_number = attempt_number + 1
                WHERE id = ?
            """, job.id)
        
        sleep(20 seconds)
```

**Why 15 seconds?**
- Executor sends heartbeat every 10 seconds
- 5-second buffer for network delays

---

### 6.4 Kafka Queues

**Three Queues:**

```
1. run_queue → Jobs ready to execute
2. retry_queue → Failed jobs to retry
3. dead_letter_queue → Jobs that exceeded max retries
```

**Why Kafka?**
- ✅ High throughput (10,000 jobs/second)
- ✅ Persistent (no message loss)
- ✅ Ordered (FIFO)

---

### 6.5 Job Consumer Service

**Responsibilities:**

1. Consume jobs from Kafka
2. Send to Executor Service
3. Update job status

**Flow:**

```python
def consume_jobs():
    while True:
        # Consume from run_queue
        job = kafka.consume("run_queue")
        
        # Send to Executor Service
        executor = get_least_loaded_executor()
        executor.execute(job)
        
        # Update status to "running"
        db.execute("""
            UPDATE job_runs
            SET status = 'running',
                start_time = NOW(),
                executor_id = ?
            WHERE job_id = ?
        """, executor.id, job.job_id)
```

---

### 6.6 Executor Service

**Responsibilities:**

1. Execute job
2. Send heartbeat every 10 seconds
3. Update final status (success/failed)

**Flow:**

```python
def execute_job(job):
    # Start execution
    publish_status(job.id, "running", start_time=now())
    
    # Heartbeat thread
    heartbeat_thread = start_heartbeat(job.id)
    
    try:
        # Execute actual job
        result = run_script(job.payload.script, job.payload.params)
        
        # Success
        publish_status(job.id, "success", end_time=now())
        
    except Exception as e:
        # Failure
        publish_status(job.id, "failed", error=str(e))
        
        # Retry logic
        if job.attempt_number < job.max_retries:
            kafka.publish("retry_queue", job)
        else:
            kafka.publish("dead_letter_queue", job)
    
    finally:
        stop_heartbeat(heartbeat_thread)


def start_heartbeat(job_id):
    def heartbeat():
        while True:
            publish_status(job_id, "running", modified_time=now())
            sleep(10 seconds)
    
    thread = Thread(target=heartbeat)
    thread.start()
    return thread


def publish_status(job_id, status, **kwargs):
    kafka.publish("status_updates", {
        "job_id": job_id,
        "status": status,
        **kwargs
    })
```

**Why heartbeat?**
- Detect dead executors
- Show real-time progress

---

### 6.7 Status Update Flow

**Flow:**

```
Executor Service
    ↓
Publish to Kafka (status_updates topic)
    ↓
Consumer Service
    ↓
Update job_runs table
```

**Consumer Service:**

```python
def consume_status_updates():
    while True:
        update = kafka.consume("status_updates")
        
        db.execute("""
            UPDATE job_runs
            SET status = ?,
                start_time = ?,
                end_time = ?,
                modified_time = ?,
                error_message = ?
            WHERE job_id = ?
        """, update.status, update.start_time, update.end_time,
             update.modified_time, update.error_message, update.job_id)
```

---

### 6.8 Retry Logic

**Three Scenarios:**

**1. Job Fails (Executor Detects):**
```
Executor catches exception
    ↓
Publish to retry_queue
    ↓
Job Consumer re-executes
```

**2. Executor Crashes:**
```
Watcher Service detects stale modified_time
    ↓
Publish to retry_queue
    ↓
Job Consumer re-executes
```

**3. Max Retries Exceeded:**
```
attempt_number >= max_retries
    ↓
Publish to dead_letter_queue
    ↓
Mark job as "failed"
```

---

### 6.9 Cancel Running Job

**Challenge:** How to stop a running job?

**Solution:** Redis Cache

**Flow:**

```
1. User clicks "Cancel"
    ↓
2. Job Service publishes to Kafka (cancel_requests)
    ↓
3. Consumer Service inserts into Redis:
    SET cancel:job_123 "true" EX 30
    ↓
4. Executor Service checks Redis every 10 seconds:
    IF redis.get("cancel:job_123"):
        stop_execution()
        publish_status("cancelled")
```

**Why Redis?**
- ✅ Fast lookup (< 1ms)
- ✅ TTL support (auto-expire after 30 seconds)

**Why TTL?**
- Prevent stale cancel requests
- Only running jobs should be affected

---

### 6.10 Run Job Immediately

**Flow:**

```
User clicks "Run Now"
    ↓
Job Service checks schedule_time
    ↓
IF schedule_time <= now() + 1 minute:
    Publish directly to run_queue (bypass Watcher)
ELSE:
    Insert into Job DB (Watcher will pick up)
```

**Why bypass Watcher?**
- Watcher scans every 20 seconds
- Immediate jobs can't wait

---

### 6.11 Edge Case: Race Condition

**Problem:**

```
Watcher scans at 5:00:00 → Fetches jobs for 5:00:00 - 5:05:00
User creates job at 5:00:10 (scheduled for 5:00:15)
Watcher scans at 5:00:20 → Fetches jobs for 5:00:20 - 5:05:20

Job scheduled at 5:00:15 is MISSED! ❌
```

**Solution:**

```python
# Job Service checks schedule_time
if schedule_time <= now() + 1 minute:
    # Bypass Watcher, publish directly to run_queue
    kafka.publish("run_queue", job)
else:
    # Let Watcher handle it
    kafka.publish("job_events", job)
```

---

### 6.12 Watcher Service State Management

**Problem:** Watcher crashes, loses track of last scan time

**Solution:** Redis Cache

**Flow:**

```python
def fetch_upcoming_jobs():
    # Get last scan time from Redis
    last_scan = redis.get("watcher:last_scan") or now()
    
    current_time = now()
    future_time = current_time + 5 minutes
    
    # Fetch jobs
    jobs = db.query("""
        SELECT * FROM jobs
        WHERE schedule_time BETWEEN ? AND ?
    """, last_scan, future_time)
    
    # Update last scan time
    redis.set("watcher:last_scan", current_time)
```

**Why Redis?**
- ✅ Persistent state
- ✅ Fast read/write

---

## 7. Complete Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    USERS                                 │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │  API Gateway    │
                └────────┬────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │   Job    │    │  Search  │    │ Watcher  │
  │ Service  │    │ Service  │    │ Service  │
  └────┬─────┘    └────┬─────┘    └────┬─────┘
       │               │                │
       │               │                │
       ▼               ▼                ▼
  ┌──────────────────────────────────────────┐
  │           Kafka Broker                    │
  │  ┌──────────┬──────────┬──────────┐      │
  │  │job_events│run_queue │retry_queue│     │
  │  └──────────┴──────────┴──────────┘      │
  └────┬─────────────────────────────────┬───┘
       │                                  │
       ▼                                  ▼
  ┌──────────┐                      ┌──────────┐
  │ Consumer │                      │   Job    │
  │ Service  │                      │ Consumer │
  │ (Write)  │                      │          │
  └────┬─────┘                      └────┬─────┘
       │                                  │
       ▼                                  ▼
  ┌──────────────────────────────────────────┐
  │           PostgreSQL (Job DB)             │
  │  ┌──────────┬──────────┐                 │
  │  │   jobs   │ job_runs │                 │
  │  └──────────┴──────────┘                 │
  └──────────────────────────────────────────┘
                         │
                         ▼
                  ┌──────────┐
                  │ Executor │
                  │ Service  │
                  └────┬─────┘
                       │
                       ▼
                  ┌──────────┐
                  │  Redis   │
                  │  Cache   │
                  └──────────┘
```

---

## 8. Database Indexing & Partitioning

### 8.1 Jobs Table

**Indexes:**
```sql
CREATE INDEX idx_job_id ON jobs(job_id);  -- Primary lookup
CREATE INDEX idx_schedule_time ON jobs(schedule_time);  -- Watcher scans
```

**Partitioning:**
```sql
-- Partition by schedule_time (monthly)
CREATE TABLE jobs_2026_01 PARTITION OF jobs
FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE TABLE jobs_2026_02 PARTITION OF jobs
FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
```

**Why partition by schedule_time?**
- Watcher Service queries by schedule_time
- Faster scans (only relevant partition)

---

### 8.2 Job Runs Table

**Indexes:**
```sql
CREATE INDEX idx_job_id ON job_runs(job_id);  -- Lookup by job
CREATE INDEX idx_status_modified ON job_runs(status, modified_time);  -- Detect dead executors
```

**Composite Index Explained:**

```sql
-- Watcher Service query
SELECT * FROM job_runs
WHERE status = 'running'
AND modified_time < '2026-01-21T12:00:00Z';

-- Uses idx_status_modified (composite index)
```

---

## 9. Interview Q&A

### Q1: Why Kafka instead of direct database writes?

**A:**

**Without Kafka:**
```
10,000 jobs/second → Direct DB writes
    ↓
Database overload ❌
```

**With Kafka:**
```
10,000 jobs/second → Kafka (buffering)
    ↓
Consumer Service → Batch writes to DB
    ↓
Database handles load ✅
```

**Benefits:**
- ✅ Decouple write operations
- ✅ Handle traffic spikes
- ✅ Prevent database overload

### Q2: Why scan every 20 seconds (not 1 second)?

**A:**

**Trade-off:**

| Interval | Database Load | Latency |
|----------|---------------|---------|
| 1 second | Very High ❌ | Very Low ✅ |
| 20 seconds | Low ✅ | Acceptable ✅ |
| 60 seconds | Very Low ✅ | High ❌ |

**Decision:** 20 seconds (balance load and latency)

**Mitigation for immediate jobs:**
- Bypass Watcher for jobs scheduled within 1 minute

### Q3: How to handle Watcher Service failure?

**A:**

**Problem:** Watcher crashes, loses state

**Solution:** Redis state management

```python
# Before crash
redis.set("watcher:last_scan", "2026-01-21T12:00:00Z")

# After restart
last_scan = redis.get("watcher:last_scan")
# Resume from last scan time
```

### Q4: How to prevent duplicate job execution?

**A:**

**Problem:** Watcher scans twice, picks same job

**Solution:** Idempotency check

```python
def fetch_upcoming_jobs():
    jobs = db.query("SELECT * FROM jobs WHERE ...")
    
    for job in jobs:
        # Check if already queued
        existing = db.query("""
            SELECT * FROM job_runs
            WHERE job_id = ? AND status = 'queued'
        """, job.job_id)
        
        if not existing:
            kafka.publish("run_queue", job)
```

### Q5: Why heartbeat every 10 seconds?

**A:**

**Purpose:**
- Detect dead executors
- Show real-time progress

**Why 10 seconds?**
- Balance between database load and detection speed
- 15-second threshold = 10s heartbeat + 5s buffer

### Q6: How to handle max retries exceeded?

**A:**

**Flow:**

```
Job fails → attempt_number++
    ↓
IF attempt_number < max_retries:
    Publish to retry_queue
ELSE:
    Publish to dead_letter_queue
    Mark job as "failed"
    Send alert to user
```

**Dead Letter Queue:**
- Manual intervention required
- Admin reviews and re-triggers

### Q7: How to cancel a scheduled (not running) job?

**A:**

**Flow:**

```
User clicks "Cancel"
    ↓
Job Service updates Job DB:
    UPDATE jobs SET status = 'cancelled'
    WHERE job_id = ?
    ↓
Watcher Service skips cancelled jobs:
    SELECT * FROM jobs
    WHERE status = 'scheduled'  -- Excludes cancelled
```

### Q8: Kafka vs Redis Sorted Set?

**A:**

**Problem with Kafka:**
```
Jobs in queue: [5:00, 5:02, 5:04, 5:06]
New job arrives: 5:01
    ↓
Inserted at end: [5:00, 5:02, 5:04, 5:06, 5:01]
    ↓
Job at 5:01 executes LAST! ❌
```

**Solution: Redis Sorted Set**

```python
# Insert with score = schedule_time
redis.zadd("job_queue", {job_id: schedule_time})

# Consume in order
jobs = redis.zrangebyscore("job_queue", 0, now())
```

**Comparison:**

| Feature | Kafka | Redis Sorted Set |
|---------|-------|------------------|
| Ordering | FIFO | Sorted by score ✅ |
| Throughput | Very High | High |
| Persistence | Disk | Memory |

**Decision:** Redis Sorted Set (better ordering)

**Alternative:** Amazon SQS (supports delayed messages)

### Q9: How to handle job logs?

**A:**

**Flow:**

```
Executor Service
    ↓
Write logs to file: /logs/job_123.log
    ↓
Upload to S3: s3://logs/job_123.log
    ↓
Update job_runs table:
    error_message = "s3://logs/job_123.log"
```

**User retrieves logs:**
```
GET /v1/jobs/{jobId}/logs
    ↓
Return S3 URL
```

### Q10: How to scale Executor Service?

**A:**

**Strategies:**

1. **Horizontal Scaling:**
   - Add more Executor instances
   - Job Consumer distributes load

2. **Resource Isolation:**
   - Docker containers per job
   - Prevent resource contention

3. **Priority Queues:**
   - High-priority jobs → Separate queue
   - Dedicated executors

---

## 10. Key Takeaways

✅ **Watcher Service:** Scans DB every 20 seconds, fetches jobs for next 5 minutes  
✅ **Kafka Queues:** run_queue, retry_queue, dead_letter_queue  
✅ **Heartbeat:** Executor sends status every 10 seconds  
✅ **Dead Executor Detection:** Check `modified_time` > 15 seconds  
✅ **Retry Logic:** Max retries, then dead letter queue  
✅ **Cancel Running Job:** Redis cache with TTL  
✅ **Immediate Jobs:** Bypass Watcher, publish directly to Kafka  
✅ **Race Condition:** Jobs within 1 minute → Direct to queue  
✅ **State Management:** Redis stores last scan time  
✅ **Ordering:** Redis Sorted Set > Kafka (for time-based ordering)  

---

## Summary

**Architecture Highlights:**
- 6+ microservices
- 2 databases (PostgreSQL for jobs, Redis for state)
- Kafka for event streaming
- Watcher Service (scans every 20 seconds)
- Executor Service (runs jobs, sends heartbeat)

**Job Lifecycle:**
```
Create → Kafka → Job DB → Watcher → Kafka Queue → Consumer → Executor → Status Updates
```

**Retry Flow:**
```
Job Fails → retry_queue → Consumer → Executor (attempt++)
Max Retries → dead_letter_queue → Manual intervention
```

**Dead Executor Detection:**
```
Executor crashes → No heartbeat → modified_time stale → Watcher detects → Re-queue
```

**Performance:**
- 10,000 jobs/second
- < 2 seconds latency
- At-least-once execution

**End of Lecture 9**
