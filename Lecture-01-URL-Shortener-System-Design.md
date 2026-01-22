# URL Shortener System Design (like Bitly)

> **Topic:** System Design  
> **Application:** URL Shortener Service  
> **Scale:** 100M Daily Active Users, 1B URLs  
> **Difficulty:** Medium

---

## 📋 Table of Contents

1. [Brief Introduction of the Application](#1-brief-introduction-of-the-application)
2. [Functional and Non-Functional Requirements](#2-functional-and-non-functional-requirements)
3. [Identification of Core Entities](#3-identification-of-core-entities)
4. [API Design](#4-api-design)
5. [High-Level Design (HLD)](#5-high-level-design-hld)
6. [Low-Level Design (LLD)](#6-low-level-design-lld)
7. [Interview-Style Design Discussion (Q&A Format)](#7-interview-style-design-discussion-qa-format)
8. [Additional Insights](#8-additional-insights)

---

<br>

## 1. Brief Introduction of the Application

### What is a URL Shortener?

A URL shortener is a service that converts long URLs into short, manageable links. For example:

- **Long URL:** `https://www.facebook.com/anindya.s.dasgupta/`
- **Short URL:** `http://bit.ly/4kyLcpo`

When users click on the short URL, they get **redirected** to the original long URL.

### Problem It Solves

1. **Easier Sharing:** Long URLs are difficult to share, especially on platforms with character limits (Twitter, SMS)
2. **Better User Experience:** Short URLs are cleaner and more memorable
3. **Tracking & Analytics:** Premium users can track how many times their links are clicked
4. **Custom Branding:** Businesses can create custom short URLs for marketing campaigns

---

## 2. Functional and Non-Functional Requirements

### 2.1 Functional Requirements

1. **Create a short URL from a long URL**
   - User provides a long URL
   - System returns a shortened version (5-7 characters)

2. **Redirect users to the original URL**
   - When user enters short URL in browser
   - System redirects to the original long URL

3. **Support Custom URLs (Premium Feature)**
   - Premium users can create custom short URLs
   - Example: Instead of `bit.ly/4kyLcpo`, user can choose `bit.ly/myFacebook`

4. **Support Custom Expiration Date (Premium Feature)**
   - Default expiration: 90 days
   - Premium users can set custom expiration dates

### 2.2 Non-Functional Requirements

1. **Low Latency**
   - URL creation: < 200ms
   - URL redirection: < 200ms
   - Users should get responses quickly

2. **High Scale**
   - Support **100 million daily active users**
   - Handle **1 billion URLs** in the system

3. **Uniqueness**
   - Every shortened URL must be **unique**
   - No two long URLs should map to the same short URL

4. **CAP Theorem - Availability > Consistency**
   - **High Availability:** System should always be accessible
   - **Eventual Consistency:** Small delay (200ms) is acceptable
   - **Why?** Users need time to share URLs anyway, so immediate consistency isn't critical
   - **Why not Consistency?** Unlike banking/ticket booking, we don't need read-after-write consistency

---

## 3. Identification of Core Entities

The main entities involved in this system:

1. **Short URL** - The shortened version of the URL
2. **Long URL** - The original URL that needs to be shortened
3. **User** - The person creating and using the URLs

---

## 4. API Design

### 4.1 API Endpoint 1: Create Short URL

```
POST /v1/url
```

**Request Body:**
```json
{
  "longURL": "https://www.facebook.com/anindya.s.dasgupta/",
  "customURL?": "myCustomURL",        // Optional (Premium)
  "expirationDate?": "2026-12-31"     // Optional (Premium)
}
```

**Response:**
```json
{
  "shortURL": "http://bit.ly/4kyLcpo"
}
```

### 4.2 API Endpoint 2: Redirect to Original URL

```
GET /v1/url/{shortURL}
```

**Response:**
```json
{
  "longURL": "https://www.facebook.com/anindya.s.dasgupta/"
}
```

**HTTP Status Code:** 301 (Permanent Redirect) or 302 (Temporary Redirect)

---

## 5. High-Level Design (HLD)

### 5.1 High-Level Design Diagram
<img width="641" height="131" alt="image" src="https://github.com/user-attachments/assets/e377931d-033e-4908-88b6-9d195f3102c8" />



```
┌──────────┐                                    ┌──────────────┐
│          │   generate(longURL)                │              │
│  Client  │──────────────────────────────────> │    Server    │
│          │                                     │              │
│          │   <──────────────────────────────  │  - Generate  │
│          │        return shortURL             │    short URL │
└──────────┘                                    └──────┬───────┘
                                                       │
     │                                                 │
     │                                                 ▼
     │                                          ┌─────────────┐
     │   redirect(shortURL)                     │             │
     │─────────────────────────────────────────>│  Database   │
     │                                           │             │
     │   <──────────────────────────────────────│  Tables:    │
     │        return longURL                     │  - User     │
                                                 │  - URL      │
                                                 └─────────────┘
```

### 5.2 Database Schema

**User Table:**
- User metadata
- Premium status
- User preferences

**URL Table:**
- `shortURL` (Primary Key)
- `longURL`
- `expirationDate`
- `customURL`

### 5.3 High-Level Design Discussion

**Q: Why do we need a database?**
- To maintain the mapping between short URLs and long URLs
- Without storage, we cannot redirect users to the original URL

**Q: What type of database should we use?**
- We'll discuss this in Low-Level Design
- For now, we just need a database that can store key-value pairs efficiently

**Q: How does the data flow work?**

**For URL Creation:**
1. Client sends long URL to server
2. Server generates short URL (using some logic - discussed in LLD)
3. Server saves mapping (shortURL → longURL) in database
4. Server returns short URL to client

**For URL Redirection:**
1. Client sends short URL to server
2. Server checks database for the mapping
3. Server retrieves the long URL
4. Server sends HTTP 301/302 redirect response
5. Client's browser redirects to the long URL

---

## 6. Low-Level Design (LLD)

### 6.1 Key Discussion Before LLD Diagram

Before we dive into the final design, let's understand **three different approaches** to generate short URLs, their pros and cons, and why we need a hybrid solution.

---

#### **Approach 1: Encryption Logic (MD5, SHA-1, Base64)**

**How it works:**
1. Take the long URL
2. Apply encryption algorithm (MD5/SHA-1)
3. Take first 6-7 characters from the encrypted string
4. Return as short URL

**Example:**
- Long URL: `https://www.facebook.com/anindya.s.dasgupta/`
- MD5 Hash: `9191c037d0c5f4408f0caf5c3cd39edd` (32 characters)
- Short URL: `9191c0` (first 6 characters)

**Problems:**

1. **Collision Risk:**
   - Two different URLs might produce the same first 6 characters
   - Example:
     - `facebook.com` → MD5 → `9191c0...`
     - `amazon.in` → MD5 → `9191c0...`
   - Both get the same short URL!

2. **High Latency Due to Collision Handling:**
   - Encrypt URL: **1ms**
   - Check database if short URL exists: **10ms**
   - Save to database: **5ms**
   - **Total: 16ms**
   
   - If collision occurs:
     - Re-encrypt with modified key: **1ms**
     - Check database again: **10ms**
     - Save to database: **5ms**
     - **Total: 32ms** (doubles with each collision!)

3. **Unpredictable Performance:**
   - For 1 billion URLs, multiple collisions are guaranteed
   - Some requests might take 10-15 collisions
   - Latency becomes unpredictable and unacceptable

**When to use:**
- Small-scale applications (< 1 million URLs)
- When uniqueness is not critical
- Prototypes and MVPs

---

#### **Approach 2: Counter-Based Approach**

**How it works:**
1. Maintain a global counter
2. For each new URL request, increment counter
3. Return counter value as the short URL identifier

**Example:**
- 1st URL → Counter = 1 → Short URL: `1`
- 2nd URL → Counter = 2 → Short URL: `2`
- 3rd URL → Counter = 3 → Short URL: `3`

**Advantages:**
- **Guaranteed Uniqueness:** Counter ensures no duplicates
- **Lower Latency:** No need to check database for duplicates
  - Encrypt: **1ms**
  - Save to DB: **5ms**
  - **Total: 6ms** (vs 16ms in Approach 1)

**Problems with Single Server:**

```
┌──────────┐
│  Server  │
│          │
│ Counter  │ ──────> Database
│  (local) │
└──────────┘
```

- Works fine for single server
- But we need to scale to 100M users!
- Single server cannot handle this load

**Problems with Multiple Servers (Microservices):**

```
                    ┌──────────────┐
                    │ Load Balancer│
                    └──────┬───────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
    ┌──────────┐     ┌──────────┐    ┌──────────┐
    │ Server 1 │     │ Server 2 │    │ Server 3 │
    │Counter=0 │     │Counter=0 │    │Counter=0 │
    └────┬─────┘     └────┬─────┘    └────┬─────┘
         │                │                │
         └────────────────┼────────────────┘
                          ▼
                    ┌──────────┐
                    │ Database │
                    └──────────┘
```

**The Problem:**
1. Request 1 → Server 1 → Counter = 0 → Returns `0`
2. Request 2 → Server 1 → Counter = 1 → Returns `1`
3. Request 3 → Server 2 → Counter = 0 → Returns `0` ❌ **DUPLICATE!**

Each server has its own local counter, causing duplicates!

**Solution: Use Redis for Global Counter**

```
                    ┌──────────────┐
                    │ Load Balancer│
                    └──────┬───────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
    ┌──────────┐     ┌──────────┐    ┌──────────┐
    │ Server 1 │     │ Server 2 │    │ Server 3 │
    └────┬─────┘     └────┬─────┘    └────┬─────┘
         │                │                │
         └────────────────┼────────────────┘
                          ▼
                    ┌──────────┐
                    │  Redis   │ ← Global Counter
                    │ (Cache)  │
                    └────┬─────┘
                         │
                         ▼
                    ┌──────────┐
                    │ Database │
                    └──────────┘
```

**How it works:**
1. All servers check Redis for counter value
2. Redis is single-threaded → Atomic operations
3. No duplicates possible!

**Latency:**
- Access Redis: **2-3ms**
- Save to DB: **5ms**
- **Total: ~7ms** ✅

**New Problem: Single Point of Failure**
- If Redis goes down, entire system fails!
- Cannot generate new short URLs

**What about Redis Cluster?**
- Multiple Redis instances
- But each instance has its own counter
- Same problem as multiple servers! ❌

**Verdict:**
- Good for mid-level interviews (SDE-2)
- Shows understanding of distributed systems
- But not production-ready for massive scale

---

#### **Approach 3: Zookeeper + Snowflake ID (Production-Ready Solution)**

This is the **hybrid approach** that combines the best of all worlds!

**What is Zookeeper?**
- Distributed coordination service for distributed applications
- Monitors all services in a microservices architecture
- Provides centralized configuration management
- Acts as a "parent monitor" for all backend services

**How Zookeeper Works:**

```
Zookeeper (Tree Structure)
    │
    ├── App1 (Ephemeral Node)
    │   ├── config
    │   └── metadata
    │
    └── App2 (Persistent Node)
        ├── config
        └── counter ← Stored here!
```

**Two Types of Nodes:**
1. **Ephemeral Node:** Deleted when application dies
2. **Persistent Node:** Never gets deleted (survives restarts)

**Zookeeper Heartbeat:**
- Sends heartbeat requests to all registered applications
- If application doesn't respond → Marks as dead
- Deletes ephemeral nodes of dead applications

---

**What is Snowflake ID?**

A **64-bit unique identifier** used in distributed systems:

```
┌─────┬──────────────────────┬────────────┬──────────────┐
│ 1bit│      41 bits         │  10 bits   │   12 bits    │
│ Sign│     Timestamp        │ Worker ID  │   Sequence   │
└─────┴──────────────────────┴────────────┴──────────────┘
```

**Components:**
1. **Sign Bit (1 bit):** Positive/Negative (always 0 for positive)
2. **Timestamp (41 bits):** Current time in milliseconds
3. **Worker ID (10 bits):** Unique ID for each server (from Zookeeper)
4. **Sequence Number (12 bits):** Local counter on each server

**Why Snowflake ID is Unique:**
- **Timestamp:** Changes every millisecond
- **Worker ID:** Unique for each server (assigned by Zookeeper)
- **Sequence:** Local counter on each server

Even if two servers generate IDs at the same millisecond, they have different Worker IDs!

---

**Complete Flow with Zookeeper + Snowflake:**

```
                    ┌──────────────┐
                    │ Load Balancer│
                    │ (Round Robin)│
                    └──────┬───────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
    ┌──────────┐     ┌──────────┐    ┌──────────┐
    │ Server 1 │     │ Server 2 │    │ Server 3 │
    │Worker:123│     │Worker:234│    │Worker:345│
    │Counter=0 │     │Counter=0 │    │Counter=0 │
    └────┬─────┘     └────┬─────┘    └────┬─────┘
         │                │                │
         └────────────────┼────────────────┘
                          │
                          ▼
                    ┌──────────┐
                    │Zookeeper │
                    │          │
                    │ Assigns  │
                    │Worker IDs│
                    └──────────┘
```

**Step-by-Step Process:**

1. **Server Startup:**
   - Server registers with Zookeeper
   - Zookeeper assigns unique Worker ID (123, 234, 345)
   - Server stores Worker ID locally

2. **URL Creation Request:**
   - Request arrives at Server 1
   - Server 1 generates Snowflake ID:
     - Sign: `0`
     - Timestamp: `1642345678901` (current time)
     - Worker ID: `123` (from Zookeeper)
     - Sequence: `0` (local counter)
   - Snowflake ID: `0000000000000001642345678901123000000000000`

3. **Encryption:**
   - Take Snowflake ID
   - Apply MD5/SHA-1 encryption
   - Take first 6-7 characters
   - This is our short URL!

4. **Why No Collisions?**
   - We control the input (Snowflake ID)
   - Snowflake ID is guaranteed unique
   - MD5 of unique input = unique output
   - No need to check database for duplicates!

**Advantages:**

✅ **Zero Latency for Uniqueness Check:**
- Everything happens locally on the server
- No Redis/Database calls needed
- **Latency: ~1ms** (just encryption)

✅ **No Single Point of Failure:**
- If Zookeeper goes down, servers keep working
- Worker IDs are already assigned
- Servers can continue generating unique IDs

✅ **Fault Tolerance:**
- If Server 3 dies, Load Balancer routes to Server 1 & 2
- Server 3 can restart and get a new Worker ID
- No data loss, no downtime

✅ **Highly Scalable:**
- Can support thousands of servers
- Each server gets unique Worker ID
- Linear scalability

---

### 6.2 Low-Level Design Diagram (Final Architecture)
<img width="798" height="280" alt="image" src="https://github.com/user-attachments/assets/efc46246-5104-4563-bd71-9d747a4b7ffa" />
<img width="660" height="332" alt="image" src="https://github.com/user-attachments/assets/5ebea70e-1ca8-4bc7-a4b7-9f333947d45e" />
<img width="627" height="290" alt="image" src="https://github.com/user-attachments/assets/9165db12-452d-4e05-ae3d-758363ad8ceb" />
<img width="573" height="223" alt="image" src="https://github.com/user-attachments/assets/b29f9fed-cf46-4989-ba6e-aad23b03f217" />





```
                                    ┌──────────────┐
                                    │  Zookeeper   │
                                    │              │
                                    │ Assigns      │
                                    │ Worker IDs:  │
                                    │  - 123       │
┌──────────┐                        │  - 234       │
│          │                        │  - 345       │
│  Client  │                        └──────────────┘
│          │                                │
└────┬─────┘                                │
     │                                      │
     │                              ┌───────▼────────┐
     │                              │ Load Balancer  │
     │                              │  (Round Robin) │
     │                              └───────┬────────┘
     │                                      │
     │                    ┌─────────────────┼─────────────────┐
     │                    │                 │                 │
     │                    ▼                 ▼                 ▼
     │            ┌───────────────┐ ┌───────────────┐ ┌───────────────┐
     │            │ Encryption    │ │ Encryption    │ │ Encryption    │
     │            │ Server 1      │ │ Server 2      │ │ Server 3      │
     │            │               │ │               │ │               │
     │            │ WorkerID: 123 │ │ WorkerID: 234 │ │ WorkerID: 345 │
     │            │ Counter: 0    │ │ Counter: 0    │ │ Counter: 0    │
     │            │               │ │               │ │               │
     │            │ Generates     │ │ Generates     │ │ Generates     │
     │            │ Snowflake ID  │ │ Snowflake ID  │ │ Snowflake ID  │
     │            └───────┬───────┘ └───────┬───────┘ └───────┬───────┘
     │                    │                 │                 │
     │                    └─────────────────┼─────────────────┘
     │                                      │
     │                                      ▼
     │                              ┌───────────────┐
     │                              │   Database    │
     │                              │  (PostgreSQL/ │
     │                              │    MySQL)     │
     │                              │               │
     │                              │  - shortURL   │
     │                              │  - longURL    │
     │                              │  - expiration │
     │                              │  - customURL  │
     │                              └───────┬───────┘
     │                                      │
     │                                      │
     │                              ┌───────▼───────┐
     │                              │   Cron Job    │
     │                              │               │
     │                              │ Runs daily to │
     │                              │ delete expired│
     │                              │ URLs          │
     │                              └───────────────┘
     │
     │
     │  redirect(shortURL)
     │──────────────────────────────────────────────────┐
     │                                                   │
     │                                                   ▼
     │                                    ┌──────────────────────┐
     │                                    │  Decryption Servers  │
     │                                    │  (5-6 instances)     │
     │                                    │                      │
     │                                    │  80% of traffic      │
     │                                    └──────────┬───────────┘
     │                                               │
     │                                               ▼
     │                                    ┌──────────────────────┐
     │                                    │   Redis Cache        │
     │                                    │                      │
     │                                    │  Key: shortURL       │
     │                                    │  Value: longURL      │
     │                                    │  TTL: expirationDate │
     │                                    │                      │
     │                                    │  Access time: 2ms    │
     │                                    └──────────┬───────────┘
     │                                               │
     │                                               │ Cache Miss
     │                                               ▼
     │                                    ┌──────────────────────┐
     │                                    │   Database           │
     │                                    │                      │
     │                                    │  Indexed on shortURL │
     │                                    │  Access time: 10ms   │
     │                                    └──────────────────────┘
     │
     │  <────────────────────────────────────────────
     │         HTTP 301/302 Redirect
     │         return longURL
```

---

### 6.3 Component Breakdown

#### **1. Encryption Servers (20% Traffic)**
- **Responsibility:** Generate short URLs from long URLs
- **Scaling:** 3 instances (can scale based on load)
- **Process:**
  1. Receive long URL from client
  2. Generate Snowflake ID (timestamp + workerID + counter)
  3. Encrypt Snowflake ID using MD5/SHA-1
  4. Take first 6-7 characters
  5. Save to database
  6. Return short URL to client

#### **2. Decryption Servers (80% Traffic)**
- **Responsibility:** Redirect users from short URL to long URL
- **Scaling:** 5-6 instances (more than encryption servers)
- **Process:**
  1. Receive short URL from client
  2. Check Redis cache first (2ms)
  3. If cache miss, check database (10ms)
  4. Return long URL with HTTP 301/302 redirect

#### **3. Load Balancer**
- **Algorithm:** Round Robin
- **Responsibility:** Distribute traffic evenly across servers

#### **4. Zookeeper**
- **Responsibility:** Assign unique Worker IDs to each server
- **Monitoring:** Send heartbeat to check server health
- **Fault Tolerance:** If server dies, remove from registry

#### **5. Redis Cache**
- **Purpose:** Reduce database load for frequently accessed URLs
- **TTL:** Set to expiration date of URL
- **Eviction:** LRU (Least Recently Used)
- **Performance:** 2ms access time

#### **6. Database (PostgreSQL/MySQL)**
- **Schema:**
  - `shortURL` (Primary Key, Indexed)
  - `longURL`
  - `expirationDate`
  - `customURL`
  - `userID`
- **Indexing:** On `shortURL` for fast lookups

#### **7. Cron Job**
- **Frequency:** Runs once daily
- **Responsibility:** Delete expired URLs from database

---

### 6.4 Latency Analysis

#### **URL Creation (Encryption):**
1. Request reaches server: **5ms**
2. Generate Snowflake ID: **<1ms** (local operation)
3. Encrypt with MD5: **1ms**
4. Save to database: **5ms**
5. **Total: ~12ms** ✅ (Well under 200ms requirement)

#### **URL Redirection (Decryption) - Cache Hit:**
1. Request reaches server: **5ms**
2. Check Redis cache: **2ms**
3. Return redirect response: **<1ms**
4. **Total: ~8ms** ✅ (Excellent!)

#### **URL Redirection (Decryption) - Cache Miss:**
1. Request reaches server: **5ms**
2. Check Redis cache: **2ms** (miss)
3. Query database: **10ms**
4. Update Redis cache: **2ms**
5. Return redirect response: **<1ms**
6. **Total: ~20ms** ✅ (Still well under 200ms)

---

### 6.5 Handling Custom URLs

**Flow for Custom URLs (Premium Feature):**

1. User requests custom short URL: `bit.ly/myFacebook`
2. Server checks database: `SELECT * FROM urls WHERE shortURL = 'myFacebook'`
3. **If exists:** Return error "Custom URL already taken"
4. **If not exists:** Create mapping and save to database
5. Return success response

**No Snowflake ID needed for custom URLs!**

---

### 6.6 Handling Expiration

**Database Level:**
- Cron job runs daily
- Query: `DELETE FROM urls WHERE expirationDate < CURRENT_DATE`

**Cache Level:**
- Redis TTL set to expiration date
- Automatic eviction when TTL expires
- **Important:** Even if URL is accessed daily, it must expire!
- TTL ensures hot keys also get deleted after expiration

---

## 7. Interview-Style Design Discussion (Q&A Format)

### **Q1: Why do we need a URL shortener in the first place?**

**A:** Long URLs are difficult to share, especially on platforms with character limits like Twitter (280 characters) or SMS. Short URLs are:
- Easier to remember
- Cleaner in appearance
- Better for marketing campaigns
- Enable tracking and analytics

---

### **Q2: Why can't we use a simple hash function like MD5 directly?**

**A:** We can, but there are problems:

1. **Collision Risk:** Two different URLs might produce the same hash (first 6 characters)
2. **Unpredictable Latency:** Need to check database for duplicates, re-hash if collision occurs
3. **Poor Performance at Scale:** For 1 billion URLs, collisions are guaranteed

**Example:**
- `facebook.com` → MD5 → `9191c037d0c5f4408f0caf5c3cd39edd`
- `amazon.in` → MD5 → `9191c037d0c5f4408f0caf5c3cd39edd`
- First 6 chars: `9191c0` (same!) ❌

---

### **Q3: Why not use a simple counter approach?**

**A:** Counter works great for single server, but fails in distributed systems:

**Problem:**
- Each server has its own local counter
- Server 1 generates ID: 0, 1, 2...
- Server 2 also generates ID: 0, 1, 2... ❌ **Duplicates!**

**Solution Attempt:** Use Redis for global counter
**New Problem:** Redis becomes single point of failure

---

### **Q4: Why is Redis a single point of failure?**

**A:** If Redis goes down:
- Cannot generate new short URLs
- Entire system stops working
- No fault tolerance

**What about Redis Cluster?**
- Each Redis instance has its own counter
- Same problem as multiple servers! ❌

---

### **Q5: How does Zookeeper + Snowflake ID solve all these problems?**

**A:** 

**Uniqueness:**
- Each server gets unique Worker ID from Zookeeper
- Snowflake ID = Timestamp + Worker ID + Local Counter
- Even if two servers generate IDs at same millisecond, Worker IDs are different

**No Single Point of Failure:**
- Zookeeper only needed during server startup
- Once Worker ID is assigned, server works independently
- If Zookeeper dies, servers continue working

**Zero Latency:**
- Everything happens locally on the server
- No Redis/Database calls for uniqueness check
- Just generate Snowflake ID and encrypt

**Fault Tolerance:**
- If server dies, Load Balancer routes to other servers
- Server can restart and get new Worker ID from Zookeeper

---

### **Q6: Which database should we use and why?**

**A:** **PostgreSQL or MySQL** (Relational Database)

**Why?**
- Simple key-value mapping (shortURL → longURL)
- ACID properties ensure data consistency
- Easy to index on `shortURL` for fast lookups
- Mature, battle-tested technology

**Why not NoSQL?**
- We don't need horizontal scalability at database level
- Caching layer (Redis) handles read load
- Relational model is simpler for this use case

**Why not MongoDB?**
- Overkill for simple key-value storage
- No complex queries or nested documents needed

---

### **Q7: Why do we need Redis cache?**

**A:** **To reduce database load and improve latency**

**Without Redis:**
- Every redirect request hits database
- Database access: 10ms
- For 100M users, database becomes bottleneck

**With Redis:**
- Cache hit: 2ms (5x faster!)
- 80% of requests are redirects (read-heavy workload)
- Database only hit on cache miss
- Reduces database load by ~80%

**Cache Strategy:** LRU (Least Recently Used)
- Popular URLs stay in cache
- Unpopular URLs get evicted

---

### **Q8: Why separate Encryption and Decryption servers?**

**A:** **Separation of Concerns + Efficient Scaling**

**Traffic Pattern:**
- 20% users create short URLs (write operation)
- 80% users use short URLs (read operation)

**Without Separation:**
- Scale both operations equally
- Waste resources on underutilized encryption servers

**With Separation:**
- Scale encryption servers: 3 instances
- Scale decryption servers: 5-6 instances
- Optimize resource utilization
- Save costs

---

### **Q9: Should we use HTTP 301 or 302 redirect?**

**A:** **Depends on the use case**

**HTTP 301 (Permanent Redirect):**
- Browser caches the redirect
- Subsequent requests don't hit our server
- **Pros:** Reduces server load
- **Cons:** Cannot track analytics (clicks, user behavior)

**HTTP 302 (Temporary Redirect):**
- Browser doesn't cache
- Every request hits our server
- **Pros:** Can track analytics, user behavior
- **Cons:** Higher server load

**Recommendation:**
- **Free users:** 301 (reduce server load)
- **Premium users:** 302 (enable analytics dashboard)

---

### **Q10: How do we handle URL expiration?**

**A:** **Two-level expiration:**

**1. Database Level:**
- Cron job runs daily
- Deletes expired URLs: `DELETE FROM urls WHERE expirationDate < CURRENT_DATE`

**2. Cache Level (Redis):**
- Set TTL (Time To Live) = expiration date
- Redis automatically evicts expired keys
- **Important:** Even hot keys (frequently accessed) must expire!

**Edge Case:**
- URL expires but still in Redis cache
- User tries to access → Cache hit → Redirect works ❌
- **Solution:** Set Redis TTL to expiration date

---

### **Q11: How does the system scale horizontally?**

**A:** 

**Encryption Servers:**
- Add more instances behind Load Balancer
- Each gets unique Worker ID from Zookeeper
- No coordination needed between servers

**Decryption Servers:**
- Add more instances behind Load Balancer
- All share same Redis cache
- Stateless servers (easy to scale)

**Database:**
- Vertical scaling (increase CPU/RAM)
- Read replicas for read-heavy workload
- Sharding if needed (shard by shortURL)

**Redis Cache:**
- Redis Cluster for horizontal scaling
- Consistent hashing for key distribution

---

### **Q12: What happens if a server crashes?**

**A:** 

**During Crash:**
- Load Balancer detects server is down (health check)
- Stops routing traffic to crashed server
- Other servers handle the load

**After Restart:**
- Server registers with Zookeeper
- Gets new Worker ID
- Starts accepting traffic again

**Data Loss?**
- No data loss! All data is in database
- Servers are stateless (except Worker ID)

---

### **Q13: What happens if Zookeeper crashes?**

**A:** 

**Impact:**
- **Existing servers:** Continue working (already have Worker IDs)
- **New servers:** Cannot start (need Worker ID from Zookeeper)

**Solution:**
- Zookeeper has built-in replication (Zookeeper ensemble)
- Typically run 3-5 Zookeeper nodes
- Quorum-based consensus (majority must agree)

**Recovery:**
- Zookeeper restarts from persistent logs
- Counter value is in persistent node (not lost)

---

### **Q14: How do we ensure CAP theorem compliance (Availability > Consistency)?**

**A:** 

**Availability:**
- Multiple server instances (no single point of failure)
- Load Balancer distributes traffic
- If one server dies, others handle requests

**Eventual Consistency:**
- Small delay (200ms) is acceptable
- User creates URL → shares with friends → friends click (time gap exists)
- No need for immediate read-after-write consistency

**Trade-off:**
- We sacrifice strong consistency for high availability
- Acceptable for this use case (unlike banking/ticket booking)

---

### **Q15: How do we handle custom URLs?**

**A:** 

**Flow:**
1. User requests: `bit.ly/myFacebook`
2. Check database: `SELECT * FROM urls WHERE shortURL = 'myFacebook'`
3. **If exists:** Return error "Already taken, try another"
4. **If not exists:** Save mapping and return success

**Race Condition:**
- Two users request same custom URL simultaneously
- **Solution:** Database constraint (UNIQUE on shortURL)
- First request succeeds, second gets error

**Premium Feature:**
- Check user's premium status before allowing custom URL
- Free users get auto-generated URLs only

---

### **Q16: What are the edge cases we need to handle?**

**A:** 

1. **Malicious URLs:**
   - Validate URL format
   - Check against blacklist (phishing, malware)
   - Rate limiting per user

2. **Very Long URLs:**
   - Set max length (e.g., 2048 characters)
   - Return error if exceeded

3. **Duplicate Long URLs:**
   - Same user creates same long URL twice
   - **Option 1:** Return existing short URL
   - **Option 2:** Create new short URL (allow duplicates)

4. **Expired URL Access:**
   - User tries to access expired URL
   - Return 404 or custom "Link expired" page

5. **Database Full:**
   - Archive old/expired URLs
   - Implement data retention policy

---

### **Q17: How do we monitor and debug the system?**

**A:** 

**Metrics to Track:**
- Request latency (p50, p95, p99)
- Error rate (4xx, 5xx)
- Cache hit ratio
- Database query time
- Server CPU/Memory usage

**Logging:**
- Log every URL creation (for analytics)
- Log every redirect (if using 302)
- Log errors and exceptions

**Alerting:**
- Alert if latency > 200ms
- Alert if error rate > 1%
- Alert if cache hit ratio < 70%
- Alert if server is down

**Tools:**
- Prometheus + Grafana (metrics)
- ELK Stack (logging)
- PagerDuty (alerting)

---

### **Q18: How do we estimate system capacity?**

**A:** 

**Given:**
- 100M daily active users
- 1B total URLs in system

**Assumptions:**
- 20% users create URLs (20M writes/day)
- 80% users use URLs (80M reads/day)

**Storage:**
- Each URL entry: ~500 bytes (shortURL + longURL + metadata)
- Total: 1B × 500 bytes = 500 GB
- With replication (3x): 1.5 TB

**Bandwidth:**
- Writes: 20M/day = 231 writes/sec
- Reads: 80M/day = 926 reads/sec
- Peak traffic (10x): 2,310 writes/sec, 9,260 reads/sec

**Servers Needed:**
- Each server handles ~1,000 req/sec
- Encryption: 3 servers (2,310 req/sec)
- Decryption: 10 servers (9,260 req/sec)

---

### **Q19: What are the security considerations?**

**A:** 

1. **Rate Limiting:**
   - Prevent abuse (spam, DDoS)
   - Limit: 10 URL creations per user per hour

2. **Authentication:**
   - Require login for URL creation
   - JWT tokens for API access

3. **URL Validation:**
   - Check URL format
   - Prevent open redirect vulnerabilities
   - Blacklist malicious domains

4. **HTTPS:**
   - Encrypt data in transit
   - Prevent man-in-the-middle attacks

5. **Database Security:**
   - Encrypt sensitive data at rest
   - Use prepared statements (prevent SQL injection)

---

### **Q20: How would you improve this system further?**

**A:** 

**1. Analytics Dashboard:**
- Track clicks, geographic location, device type
- Premium feature for business users

**2. QR Code Generation:**
- Generate QR code for each short URL
- Useful for print media, posters

**3. Link Preview:**
- Show preview of destination URL
- Increase user trust

**4. Custom Domains:**
- Allow users to use their own domain (e.g., `mycompany.co/xyz`)
- Premium feature

**5. A/B Testing:**
- Create multiple short URLs for same long URL
- Track which performs better

**6. API Rate Limiting:**
- Implement token bucket algorithm
- Prevent abuse

**7. Geo-based Routing:**
- Redirect users to region-specific URLs
- Example: US users → amazon.com, India users → amazon.in

---

## 8. Additional Insights

### **Why This Design is Production-Ready**

1. **Scalability:** Horizontal scaling of stateless servers
2. **Fault Tolerance:** No single point of failure
3. **Low Latency:** Redis caching + local Snowflake ID generation
4. **Uniqueness:** Guaranteed by Snowflake ID
5. **Monitoring:** Comprehensive metrics and alerting
6. **Security:** Rate limiting, authentication, validation

### **Key Takeaways**

✅ **Always discuss trade-offs** in system design interviews  
✅ **Start simple, then optimize** (Approach 1 → 2 → 3)  
✅ **Understand CAP theorem** and when to prioritize Availability vs Consistency  
✅ **Separation of concerns** (Encryption vs Decryption servers)  
✅ **Caching is critical** for read-heavy workloads  
✅ **Snowflake ID** is the gold standard for distributed unique ID generation  

### **Common Mistakes to Avoid**

❌ Using MD5 directly without handling collisions  
❌ Not separating read and write operations  
❌ Ignoring cache expiration (TTL)  
❌ Not discussing monitoring and alerting  
❌ Forgetting to index database on shortURL  
❌ Not handling edge cases (expired URLs, malicious URLs)  

---

## Summary

This URL Shortener system design demonstrates:
- **Three different approaches** with increasing complexity and reliability
- **Hybrid solution** combining Zookeeper + Snowflake ID + MD5 encryption
- **Microservices architecture** with separation of concerns
- **Caching strategy** to reduce database load
- **Fault tolerance** with no single point of failure
- **Interview-ready discussion** covering all aspects of system design

**Final Architecture:**
- Load Balancer (Round Robin)
- Encryption Servers (3 instances, 20% traffic)
- Decryption Servers (5-6 instances, 80% traffic)
- Zookeeper (Worker ID assignment)
- Redis Cache (2ms latency)
- PostgreSQL/MySQL Database (indexed on shortURL)
- Cron Job (daily cleanup of expired URLs)

**Performance:**
- URL Creation: ~12ms
- URL Redirection (cache hit): ~8ms
- URL Redirection (cache miss): ~20ms

All well under the 200ms latency requirement! ✅

---

**End of Notes**
