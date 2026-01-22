# Lecture 14: Leaderboard System Design (Top K Trending)

> **Lecture:** 14  
> **Topic:** System Design  
> **Application:** Leaderboard / Top K Trending (Gaming, Spotify, YouTube)  
> **Scale:** 1 Million Requests/Second, Billions of Entities  
> **Difficulty:** Hard  
> **Previous:** [[Lecture-13-Social-Media-Platform-System-Design|Lecture 13: Social Media Platform]]  
> **Related:** LeetCode 347: Top K Frequent Elements

---

## 📋 Table of Contents

1. [Brief Introduction](#1-brief-introduction)
2. [Functional and Non-Functional Requirements](#2-functional-and-non-functional-requirements)
3. [Core Entities](#3-core-entities)
4. [API Design](#4-api-design)
5. [High-Level Design (HLD)](#5-high-level-design-hld)
6. [Low-Level Design (LLD)](#6-low-level-design-lld)
7. [Real-Time Updates (WebSocket)](#7-real-time-updates-websocket)
8. [Database Comparison](#8-database-comparison)
9. [Interview Q&A](#9-interview-qa)
10. [Key Takeaways](#10-key-takeaways)

---

<br>

## 1. Brief Introduction

### What is a Leaderboard System?

A **leaderboard system** fetches the top K items from billions of records based on some metric (score, views, likes, plays).

**Use Cases:**
- **Gaming:** Top 100 players globally
- **YouTube:** Top trending videos (24h, 7d, 30d)
- **Spotify:** Most played songs this week
- **E-commerce:** Top sellers this month

**Key Challenge:** Fetch top K from **billions** of records in **< 100ms**

---

## 2. Functional and Non-Functional Requirements

### 2.1 Functional Requirements

1. **Insert/Update/Delete Data**
   - Insert score for player
   - Update view count for video
   - Delete obsolete data

2. **Query Top K List**
   - Fetch top 100 players
   - Fetch top 50 trending videos

3. **Filter by Time Period**
   - Last hour
   - Last 24 hours
   - Last 30 days
   - Last 90 days
   - All-time

4. **Real-Time Updates** (For Leaderboards)
   - Live score updates during gaming tournament
   - Push updates to clients via WebSocket

### 2.2 Non-Functional Requirements

1. **Scale**
   - **1 million requests/second** (insertions!)
   - **Billions of entities** (songs, videos, players)

2. **CAP Theorem**
   - **Highly Available (AP):** Eventual consistency acceptable
   
   **Why?**
   - ✅ If trending video moves from #2 → #1, delay of 1-2 mins is OK
   - ✅ Downtime = Bad user experience

3. **Latency**
   - **Fetch top K:** < 100ms
   - **Insert/Update:** < 500ms

4. **Accuracy**
   - **Accurate results** (not probabilistic)
   - Exact top K, not "probably top K"

---

## 3. Core Entities

1. **Score/View/Like** - Metric for ranking
2. **Entity** - Player, video, song
3. **Time Frame** - Hour, day, week, month, all-time

---

## 4. API Design

### 4.1 Insert/Update Score

```
POST /v1/leaderboard/{leaderboardId}/score
```

**Request Body:**
```json
{
  "entityId": "player_123",  // or video_id, song_id
  "score": 9850,  // or views, likes
  "timestamp": "2026-01-21T10:30:00Z"
}
```

---

### 4.2 Get Top K

```
GET /v1/leaderboard/{leaderboardId}?window={time}&region={region}&k={k}&page={page}
```

**Query Params:**
- `window`: `1h`, `24h`, `7d`, `30d`, `90d`, `all`
- `region`: `US`, `IN`, `UK`, etc.
- `k`: 1 to 10,000 (upper limit)
- `page`: For pagination

**Response:**
```json
{
  "leaderboard": [
    {"rank": 1, "entityId": "player_456", "name": "ProGamer", "score": 10500},
    {"rank": 2, "entityId": "player_789", "name": "ElitePlayer", "score": 10200},
    {"rank": 3, "entityId": "player_101", "name": "Champion", "score": 9850}
  ],
  "totalPages": 100
}
```

**Support:**
- ✅ REST API (for trending videos/songs)
- ✅ WebSocket (for real-time leaderboard updates)

---

### 4.3 Get User Rank (+ Neighbors)

```
GET /v1/leaderboard/{leaderboardId}/rank?userId={userId}&window={time}&k={k}
```

**Use Case:**
- User wants to see their rank + nearby players

**Example:**
```
User is rank #523
k=5
    ↓
Return:
- Rank #521
- Rank #522
- Rank #523 (User) ← Highlighted
- Rank #524
- Rank #525
```

---

## 5. High-Level Design (HLD)

### 5.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    USERS                                 │
└────────────────────────┬────────────────────────────────┘
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
  │  Score   │    │ Ranking  │    │          │
  │ Service  │    │ Service  │    │          │
  └────┬─────┘    └────┬─────┘    └──────────┘
       │               │
       ▼               ▼
  ┌──────────┐    ┌──────────┐
  │ Score DB │    │ Score DB │
  └──────────┘    └──────────┘
```

**Flow:**
1. **Score Service:** Insert/update scores
2. **Ranking Service:** Fetch top K

⚠️ **Problem:** This naive approach doesn't scale!

---

## 6. Low-Level Design (LLD)

### 6.1 Problem Statement (Like LeetCode 347)

**LeetCode 347: Top K Frequent Elements**

```
Input: nums = [1,1,1,2,2,3], k = 2
Output: [1,2]

Explanation:
1 appears 3 times ← Most frequent
2 appears 2 times ← 2nd most frequent
3 appears 1 time
```

**DSA Solution:**
```java
// Use Priority Queue (Heap)
PriorityQueue<Map.Entry<Integer, Integer>> heap = 
    new PriorityQueue<>((a, b) -> b.getValue() - a.getValue());

// Add all (number, frequency) pairs
heap.addAll(frequencyMap.entrySet());

// Pop top K
List<Integer> result = new ArrayList<>();
for (int i = 0; i < k; i++) {
    result.add(heap.poll().getKey());
}
```

**Key Concept:** Use **Heap** to get top K in O(N log K)

**System Design Equivalent:** Use **Redis Sorted Set** (acts like a heap!)

---

## 6.2 Solution 1: Redis Sorted Set (Small Scale)

### **Architecture:**

```
┌─────────────────────────────────────────────────────────┐
│                    USER                                  │
│         POST /v1/leaderboard/score                       │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │  Score Service  │
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │     Kafka       │
                │  (score_events) │
                └────────┬────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │    DB    │    │  Redis   │    │          │
  │ Consumer │    │ Consumer │    │          │
  └────┬─────┘    └────┬─────┘    └──────────┘
       │               │
       ▼               ▼
  ┌──────────┐    ┌──────────┐
  │ Score DB │    │  Redis   │
  │(Cassandra│    │  ZSet    │
  └──────────┘    │ (Sorted) │
                  └────┬─────┘
                       │
                       ▼
                  ┌──────────┐
                  │ Ranking  │
                  │ Service  │
                  └──────────┘
```

---

### **Redis Sorted Set (ZSet) Explained:**

**Key:** `leaderboard:global:alltime`

**Data Structure:**
```
ZADD leaderboard:global:alltime 10500 player_456
ZADD leaderboard:global:alltime 10200 player_789
ZADD leaderboard:global:alltime 9850 player_101
```

**Fetch Top K:**
```
ZREVRANGE leaderboard:global:alltime 0 99  // Top 100
```

**Properties:**
- ✅ Auto-sorted by score
- ✅ O(log N) insert
- ✅ O(K) fetch top K

---

### **Problem 1: All-Time Data Only**

```
Key: leaderboard:global:alltime

Problem: Only stores all-time data
User wants: Last 24h, 7d, 30d, 90d
```

---

### **Solution: Multiple Keys (Per Time Frame)**

```
leaderboard:IN:alltime
leaderboard:IN:90d
leaderboard:IN:30d
leaderboard:IN:24h
leaderboard:IN:1h

leaderboard:US:alltime
leaderboard:US:90d
...
```

**Redis Consumer Logic:**
```java
void onScoreEvent(ScoreEvent event) {
    String entity = event.getEntityId();
    int score = event.getScore();
    String region = event.getRegion();
    long timestamp = event.getTimestamp();
    
    // Add to all time frames
    redis.zadd("leaderboard:" + region + ":alltime", score, entity);
    redis.zadd("leaderboard:" + region + ":90d", score, entity);
    redis.zadd("leaderboard:" + region + ":30d", score, entity);
    redis.zadd("leaderboard:" + region + ":24h", score, entity);
    redis.zadd("leaderboard:" + region + ":1h", score, entity);
    
    // Set TTL for time-based keys
    redis.expire("leaderboard:" + region + ":90d", 90 * 24 * 3600);
    redis.expire("leaderboard:" + region + ":30d", 30 * 24 * 3600);
    redis.expire("leaderboard:" + region + ":24h", 24 * 3600);
    redis.expire("leaderboard:" + region + ":1h", 3600);
}
```

---

### **Ranking Service:**

```java
List<RankEntry> getTopK(String region, String window, int k) {
    String key = "leaderboard:" + region + ":" + window;
    
    // Fetch top K (descending order)
    Set<Tuple> result = redis.zrevrangeWithScores(key, 0, k - 1);
    
    List<RankEntry> rankings = new ArrayList<>();
    int rank = 1;
    for (Tuple tuple : result) {
        rankings.add(new RankEntry(rank++, tuple.getElement(), tuple.getScore()));
    }
    
    return rankings;
}
```

---

### **Problem 2: Scaling**

```
Single Redis instance:
- Billions of entities
- Hundreds of keys
- Single point of failure ❌
```

---

### **Solution: Redis Cluster (Partition by Region)**

```
Redis Cluster:
  Node 1 → US data
  Node 2 → IN data
  Node 3 → UK data
  ...
```

**Benefits:**
- ✅ Horizontal scaling
- ✅ No single point of failure

**Drawback:**
- Still limited by memory

---

### **Pros & Cons (Solution 1):**

| Pros | Cons |
|------|------|
| ✅ Super fast (< 10ms) | ❌ Limited by Redis memory |
| ✅ Real-time updates | ❌ Single point of failure (without cluster) |
| ✅ Simple implementation | ❌ Not durable (restart = data loss) |

**When to Use:**
- Small to medium scale (< 100M entities)
- Single region deployment
- Gaming leaderboards

---

## 6.3 Solution 2: Precomputed Aggregation (Large Scale)

### **Architecture:**

```
┌─────────────────────────────────────────────────────────┐
│                    USER                                  │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │  Score Service  │
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │     Kafka       │
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │   DB Consumer   │
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │    Score DB     │
                │   (Cassandra)   │
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │  Apache Flink   │
                │   (or Spark)    │
                │  (Batch Process)│
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │  Aggregated DB  │
                │  (InfluxDB/     │
                │   TimeSeries)   │
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │  Cache Layer    │
                │    (Redis)      │
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │ Ranking Service │
                └─────────────────┘
```

---

### **Flow:**

```
Step 1: Score events → Kafka
    ↓
Step 2: DB Consumer → Score DB (Cassandra)
    ↓
Step 3: Apache Flink reads Score DB
    - Group by region, time window
    - Calculate top K
    - Run every 5 minutes (or hourly)
    ↓
Step 4: Store in Aggregated DB (InfluxDB)
    ↓
Step 5: Ranking Service queries Aggregated DB
```

---

### **Score DB Schema (Cassandra):**

```sql
CREATE TABLE scores (
    entity_id UUID,
    score BIGINT,
    region VARCHAR(10),
    timestamp TIMESTAMP,
    PRIMARY KEY ((region), timestamp, entity_id)
);

-- Partition by region for horizontal scaling
-- Clustered by timestamp for time-based queries
```

**Why Cassandra?**
- ✅ Write-optimized (1M writes/second)
- ✅ Horizontal scaling
- ✅ High availability

---

### **Apache Flink Aggregation:**

```java
// Pseudo-code
DataStream<ScoreEvent> scoreStream = kafka.consume("score_events");

// Group by region and 1-hour window
scoreStream
    .keyBy(ScoreEvent::getRegion)
    .window(TumblingEventTimeWindows.of(Time.hours(1)))
    .process(new TopKAggregator(100))
    .addSink(aggregatedDB);

// TopKAggregator keeps top 100 in-memory using heap
class TopKAggregator extends ProcessWindowFunction<ScoreEvent, TopKResult, String, TimeWindow> {
    @Override
    public void process(String region, Context context, Iterable<ScoreEvent> events, Collector<TopKResult> out) {
        PriorityQueue<ScoreEvent> heap = new PriorityQueue<>(100, (a, b) -> b.score - a.score);
        
        for (ScoreEvent event : events) {
            heap.offer(event);
            if (heap.size() > 100) {
                heap.poll();  // Remove lowest
            }
        }
        
        // Output top 100
        out.collect(new TopKResult(region, context.window(), new ArrayList<>(heap)));
    }
}
```

---

### **Aggregated DB Schema (InfluxDB - Time-Series):**

```sql
-- InfluxDB (NoSQL, optimized for time-series)

measurement: top_k_rankings

tags:
  - region: US, IN, UK
  - window: 1h, 24h, 7d, 30d

fields:
  - rank: 1, 2, 3, ...
  - entity_id: player_123
  - score: 10500

time: 2026-01-21T10:00:00Z
```

**Query Example:**
```sql
SELECT entity_id, score
FROM top_k_rankings
WHERE region = 'US'
AND window = '24h'
AND time > now() - 24h
ORDER BY rank
LIMIT 100
```

---

### **Cache Layer (Redis):**

```
Key: topk:US:24h
Value: [
  {rank: 1, entity: "player_456", score: 10500},
  {rank: 2, entity: "player_789", score: 10200},
  ...
]

TTL: 5 minutes (refresh from Aggregated DB)
```

---

### **Ranking Service:**

```java
List<RankEntry> getTopK(String region, String window, int k) {
    // Try cache first
    String cacheKey = "topk:" + region + ":" + window;
    List<RankEntry> cached = redis.get(cacheKey);
    
    if (cached != null) {
        return cached.subList(0, Math.min(k, cached.size()));
    }
    
    // Cache miss: Query Aggregated DB
    List<RankEntry> result = influxDB.query(
        "SELECT * FROM top_k_rankings " +
        "WHERE region = '" + region + "' AND window = '" + window + "' " +
        "ORDER BY rank LIMIT " + k
    );
    
    // Cache for 5 minutes
    redis.setex(cacheKey, 300, result);
    
    return result;
}
```

---

### **Pros & Cons (Solution 2):**

| Pros | Cons |
|------|------|
| ✅ Scalable (billions of entities) | ❌ Not real-time (lag from Flink) |
| ✅ Durable (persisted in DB) | ❌ Complex architecture |
| ✅ Cost-effective (no huge Redis) | ❌ Batch processing delay |

**When to Use:**
- Large scale (> 1B entities)
- Multiple regions
- Real-time updates not critical (YouTube trending)

---

## 6.4 Solution 3: Hybrid (Best of Both Worlds!) ⭐

### **Architecture:**

```
┌─────────────────────────────────────────────────────────┐
│                    USER                                  │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │  Score Service  │
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │     Kafka       │
                └────────┬────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │    DB    │    │  Redis   │    │          │
  │ Consumer │    │ Consumer │    │          │
  └────┬─────┘    └────┬─────┘    └──────────┘
       │               │
       ▼               ▼
  ┌──────────┐    ┌──────────┐
  │ Score DB │    │  Redis   │
  │(Cassandra│    │   ZSet   │
  └────┬─────┘    │  (TTL!)  │
       │          └────┬─────┘
       ▼               │
  ┌──────────┐         │
  │  Flink   │         │
  └────┬─────┘         │
       │               │
       ▼               │
  ┌──────────┐         │
  │Aggregated│         │
  │   DB     │         │
  │(InfluxDB)│         │
  └────┬─────┘         │
       │               │
       └───────┬───────┘
               │
               ▼
        ┌──────────┐
        │ Ranking  │
        │ Service  │
        └──────────┘
```

---

### **Key Concept: Hot + Cold Data**

**Hot Data (Recent):**
- Last 1 hour
- Last 24 hours
- Last 7 days
- Stored in **Redis ZSet** (with TTL)

**Cold Data (Historical):**
- Last 30 days
- Last 90 days
- All-time
- Stored in **Aggregated DB**

---

### **Redis ZSet with TTL:**

```java
void onScoreEvent(ScoreEvent event) {
    String entity = event.getEntityId();
    int score = event.getScore();
    String region = event.getRegion();
    
    // Only store hot data (recent time windows)
    redis.zadd("leaderboard:" + region + ":1h", score, entity);
    redis.expire("leaderboard:" + region + ":1h", 3600);  // 1 hour TTL
    
    redis.zadd("leaderboard:" + region + ":24h", score, entity);
    redis.expire("leaderboard:" + region + ":24h", 86400);  // 24 hour TTL
    
    redis.zadd("leaderboard:" + region + ":7d", score, entity);
    redis.expire("leaderboard:" + region + ":7d", 7 * 86400);  // 7 day TTL
}
```

**Benefit:** Redis doesn't store all-time data (saves memory!)

---

### **Ranking Service (Hybrid Query):**

```java
List<RankEntry> getTopK(String region, String window, int k) {
    // Hot data windows → Query Redis
    if (window.equals("1h") || window.equals("24h") || window.equals("7d")) {
        String key = "leaderboard:" + region + ":" + window;
        
        // Try Redis
        if (redis.exists(key)) {
            return redis.zrevrangeWithScores(key, 0, k - 1);
        }
        
        // Redis miss (expired) → Fallback to Aggregated DB
        return queryAggregatedDB(region, window, k);
    }
    
    // Cold data windows → Query Aggregated DB
    return queryAggregatedDB(region, window, k);
}

List<RankEntry> queryAggregatedDB(String region, String window, int k) {
    // Check cache first
    String cacheKey = "topk:" + region + ":" + window;
    List<RankEntry> cached = redis.get(cacheKey);
    
    if (cached != null) {
        return cached;
    }
    
    // Query InfluxDB
    List<RankEntry> result = influxDB.query(...);
    
    // Cache for 5 mins
    redis.setex(cacheKey, 300, result);
    
    return result;
}
```

---

### **Flow Diagram:**

```
User requests: "Top 100 videos, last 24h, US"
    ↓
Ranking Service checks: window = 24h (hot data)
    ↓
Query Redis: leaderboard:US:24h
    ↓
IF EXISTS: Return from Redis (< 10ms) ✅
    ↓
IF NOT EXISTS (expired):
    ↓
    Fallback to Aggregated DB
    ↓
    Check cache (Redis)
    ↓
    IF MISS: Query InfluxDB → Cache → Return
```

---

### **Backup & Recovery:**

**Scenario:** Redis crashes

```
Redis crashes
    ↓
All ZSet data lost
    ↓
Ranking Service falls back to Aggregated DB
    ↓
Redis restarts
    ↓
Backfill from Aggregated DB:
    - Query last 24h data
    - Repopulate Redis ZSets
    ↓
System restored
```

---

### **Pros & Cons (Solution 3):**

| Pros | Cons |
|------|------|
| ✅ Real-time for hot data | ❌ Most complex architecture |
| ✅ Scalable for cold data | ❌ Need to manage TTLs |
| ✅ Durable (Aggregated DB backup) | ❌ Higher operational cost |
| ✅ Memory efficient (Redis only hot data) | |

**When to Use:**
- ✅ Global scale (YouTube, Spotify)
- ✅ Need real-time + historical
- ✅ Multiple regions

---

## 7. Real-Time Updates (WebSocket)

### **For Gaming Leaderboards:**

```
User opens leaderboard page
    ↓
Establish WebSocket connection
    ↓
Subscribe to: leaderboard:global:live
    ↓
On score update:
    - Redis Consumer updates ZSet
    - Publish to Redis Pub/Sub
    ↓
WebSocket Server receives event
    ↓
Push to subscribed clients
    ↓
Client sees live rank changes
```

---

### **WebSocket Server:**

```java
// Pseudo-code
class LeaderboardWebSocketServer {
    
    void onConnect(WebSocketSession session, String leaderboardId) {
        // Subscribe to Redis Pub/Sub
        redis.subscribe("leaderboard:" + leaderboardId + ":updates", (message) -> {
            // Push to client
            session.send(message);
        });
    }
    
    void onMessage(WebSocketSession session, String message) {
        // Handle client messages (if needed)
    }
    
    void onDisconnect(WebSocketSession session) {
        // Unsubscribe
        redis.unsubscribe(...);
    }
}
```

---

## 8. Database Comparison

| Database | Use Case | Why |
|----------|----------|-----|
| **Cassandra** | Score DB | Write-heavy, horizontal scaling |
| **InfluxDB** | Aggregated DB | Time-series queries, compression |
| **Redis ZSet** | Hot rankings | In-memory, auto-sorted, fast |
| **PostgreSQL** | Metadata | ACID, relational |

---

## 9. Interview Q&A

### Q1: Why not just query database with ORDER BY?

**A:**

**Naive Query:**
```sql
SELECT * FROM scores
WHERE region = 'US'
ORDER BY score DESC
LIMIT 100;
```

**Problem:**
```
Billions of rows
    ↓
Full table scan + sort
    ↓
Takes 10+ seconds ❌
```

**Solution:**
- Precompute (Flink + Aggregated DB)
- Or use Redis ZSet (auto-sorted)

### Q2: Why Redis Sorted Set for real-time?

**A:**

**Properties:**
- ✅ Auto-sorted by score
- ✅ O(log N) insert
- ✅ O(K) fetch top K
- ✅ In-memory (< 10ms)

**Alternative:** PostgreSQL with index
- ❌ Still slower (disk I/O)
- ❌ Locks on updates

### Q3: How to handle tie-breakers?

**A:**

**Scenario:**
```
Player A: 9,850 points
Player B: 9,850 points
Who ranks higher?
```

**Solution 1: Timestamp (earlier = higher rank)**
```java
// Store score as: score + (1 / timestamp)
double adjustedScore = score + (1.0 / timestamp);
redis.zadd(key, adjustedScore, entity);
```

**Solution 2: Lexicographical (alphabetical order)**
```java
redis.zadd(key, score, entity + ":" + timestamp);
```

### Q4: How to get user's rank efficiently?

**A:**

```java
long getRank(String leaderboardKey, String userId) {
    // O(log N)
    Long rank = redis.zrevrank(leaderboardKey, userId);
    return rank != null ? rank + 1 : -1;  // +1 because 0-indexed
}
```

**Get rank + neighbors:**
```java
List<RankEntry> getRankWithNeighbors(String key, String userId, int k) {
    Long rank = redis.zrevrank(key, userId);
    
    if (rank == null) return Collections.emptyList();
    
    // Get k neighbors above and below
    long start = Math.max(0, rank - k);
    long end = rank + k;
    
    return redis.zrevrangeWithScores(key, start, end);
}
```

### Q5: How to handle multiple regions efficiently?

**A:**

**Option 1: Separate keys**
```
leaderboard:US:24h
leaderboard:IN:24h
leaderboard:UK:24h
```

**Option 2: Redis Cluster (partition by region)**
```
Cluster Node 1 → US data
Cluster Node 2 → IN data
Cluster Node 3 → UK data
```

### Q6: What if Redis runs out of memory?

**A:**

**Solutions:**

1. **Enable eviction policy:**
   ```
   maxmemory-policy allkeys-lru  # Evict least recently used
   ```

2. **Use TTLs (Solution 3):**
   - Only store recent data
   - Historical → Aggregated DB

3. **Horizontal scaling:**
   - Redis Cluster
   - Partition by region

### Q7: How often to run Flink aggregation?

**A:**

**Trade-off:**
- More frequent = More real-time, More compute cost
- Less frequent = Less real-time, Less compute cost

**Recommendation:**
```
Hot data (1h, 24h): Every 5 minutes
Warm data (7d): Every 30 minutes
Cold data (30d, 90d): Every 6 hours
```

### Q8: How to prevent duplicate score submissions?

**A:**

**Idempotency Key:**
```java
void submitScore(String entityId, int score, String idempotencyKey) {
    // Check if already processed
    if (redis.exists("idempotency:" + idempotencyKey)) {
        return;  // Duplicate, ignore
    }
    
    // Process score
    kafka.produce("score_events", new ScoreEvent(entityId, score));
    
    // Mark as processed (TTL 24h)
    redis.setex("idempotency:" + idempotencyKey, 86400, "processed");
}
```

### Q9: How to handle historical queries (e.g., "Top 100 in Jan 2025")?

**A:**

**Scenario:** User wants "Top 100 players in January 2025"

**Solution:**
```
Aggregated DB stores snapshots:
    - 2025-01-01 → 2025-01-31
    
Query InfluxDB:
    SELECT * FROM top_k_rankings
    WHERE time >= '2025-01-01'
    AND time <= '2025-01-31'
    ORDER BY rank
    LIMIT 100
```

### Q10: Security considerations?

**A:**

1. **Rate Limiting:** Prevent spam score submissions
2. **Authentication:** Verify user identity
3. **Anti-Cheat:** Detect abnormal score patterns
4. **Encryption:** Secure score transmission
5. **Audit Log:** Track all score changes

---

## 10. Key Takeaways

✅ **Three Solutions:** Redis ZSet (small), Precomputed (large), Hybrid (best)  
✅ **Redis Sorted Set:** Acts like heap, O(log N) insert, O(K) fetch  
✅ **Hot vs Cold Data:** Recent (Redis), Historical (Aggregated DB)  
✅ **Apache Flink:** Batch process for aggregation  
✅ **TTL Strategy:** Only cache recent data (save memory)  
✅ **Cassandra:** Write-heavy score ingestion  
✅ **InfluxDB:** Time-series aggregated rankings  
✅ **WebSocket:** Real-time leaderboard updates  
✅ **Kafka:** Buffer for 1M writes/second  
✅ **Hybrid = Real-time + Scalable + Durable**  

---

## Summary

**Three Approaches:**

| Approach | Best For | Latency | Durability | Scale |
|----------|----------|---------|------------|-------|
| **Redis ZSet** | Gaming (small) | < 10ms | Low | 100M |
| **Precomputed** | YouTube trending | ~1s | High | 10B+ |
| **Hybrid** | Global apps | < 100ms | High | 10B+ |

**Hybrid Flow:**
```
Hot data (1h, 24h, 7d) → Redis ZSet (with TTL)
Cold data (30d, 90d, all) → InfluxDB (Flink aggregated)
Ranking Service → Check Redis first → Fallback to DB
```

**Key Technologies:**
- Kafka (buffer)
- Cassandra (score storage)
- Redis ZSet (real-time rankings)
- Apache Flink (aggregation)
- InfluxDB (time-series)
- WebSocket (live updates)

**Performance:**
- Fetch top K: < 100ms
- Insert score: < 500ms
- 1M requests/second
- Billions of entities

**End of Lecture 14**
