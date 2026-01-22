# Lecture 4: OTT Platform System Design (like Netflix)

> **Lecture:** 04  
> **Topic:** System Design  
> **Application:** OTT Platform (Netflix, Amazon Prime, Hotstar)  
> **Scale:** 200M Users, 10K Videos  
> **Difficulty:** Hard  
> **Previous:** [[Lecture-03-E-Commerce-Platform-System-Design|Lecture 3: E-Commerce Platform]]

---

## 📋 Table of Contents

1. [Brief Introduction](#1-brief-introduction)
2. [Functional and Non-Functional Requirements](#2-functional-and-non-functional-requirements)
3. [Core Entities](#3-core-entities)
4. [API Design](#4-api-design)
5. [High-Level Design (HLD)](#5-high-level-design-hld)
6. [Low-Level Design (LLD)](#6-low-level-design-lld)
7. [Streaming Protocols](#7-streaming-protocols)
8. [Complete Architecture Diagram](#8-complete-architecture-diagram)
9. [Interview-Style Q&A](#9-interview-style-qa)
10. [Key Takeaways](#10-key-takeaways)

---

<br>

## 1. Brief Introduction

### What is an OTT Platform?

**OTT (Over-The-Top)** platforms are video streaming services that deliver content directly to viewers over the internet.

**Examples:** Netflix, Amazon Prime Video, Hotstar, Disney+, HBO Max

### Problem It Solves

1. **On-Demand Viewing:** Watch anytime, anywhere
2. **No Buffering:** Smooth playback experience
3. **Adaptive Quality:** Adjusts to internet speed
4. **Multi-Device Support:** Watch on phone, TV, laptop
5. **Subscription Management:** Easy payment and access control

### Why This Question is "Lucky"

⚠️ **Important Context:**

If you get this question in an interview, **you're lucky!** Here's why:

- **Video processing** is a specialized domain
- **Not expected** to know all encoding details
- **High-level understanding** is sufficient
- Focus on **architecture**, not video codecs

**What interviewer expects:**
- ✅ Overall system architecture
- ✅ How videos are stored and served
- ✅ How to prevent buffering
- ❌ Deep video encoding knowledge (unless you're applying for video team)

---

## 2. Functional and Non-Functional Requirements

### 2.1 Functional Requirements

1. **User Registration & Subscription**
   - User can create account
   - Choose subscription plan (monthly, yearly)
   - Make payment securely

2. **Search Movies/Shows**
   - Search by title, genre, actor
   - Browse categories
   - View trending content

3. **Watch Videos**
   - Play videos in multiple resolutions (480p, 720p, 1080p, 4K)
   - Adaptive bitrate streaming (quality adjusts to internet speed)
   - Resume from where you left off

### 2.2 Out of Scope

❌ **Recommendation Engine**
- Requires ML/AI knowledge
- Complex algorithm (collaborative filtering, content-based)
- Keep it out of scope (mention to interviewer)

❌ **Video Upload by Users**
- Netflix is **not** YouTube
- Only backend team uploads content
- No user-generated content

### 2.3 Non-Functional Requirements

1. **Scale**
   - **200 million users** globally
   - **10,000 videos** (each ~1 hour)
   - Unlike YouTube (billions of videos), Netflix has curated content

2. **CAP Theorem - Availability > Consistency**

   **High Availability (Priority):**
   - ✅ Platform must be **always accessible**
   - ✅ Zero downtime (users expect 24/7 access)
   - ✅ Videos should load quickly

   **Consistency (Lower Priority):**
   - ✅ Payment & Subscription (must be consistent)
   - ❌ Video availability (eventual consistency okay)
   
   **Why?**
   - Netflix is **not real-time** (like YouTube live)
   - Backend team uploads videos → processes → publishes
   - Hours/days gap between upload and publish is acceptable

3. **Low Latency - Zero Buffering**
   - **Most critical requirement!**
   - Video should play smoothly
   - Minimal buffering (< 2 seconds initial load)
   - Adaptive quality based on bandwidth

---

## 3. Core Entities

1. **User** - Account holder
2. **User Metadata** - Subscription status, expiry date
3. **Video** - Actual video files
4. **Video Metadata** - Title, description, genre, duration
5. **Thumbnails** - Images for video previews

---

## 4. API Design

### 4.1 User Registration

```
POST /v1/users/register
```

**Request Body:**
```json
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "hashed_password"
}
```

**Additional APIs** (mention verbally):
- `POST /v1/users/login` - Login
- `POST /v1/users/logout` - Logout
- `PUT /v1/users/profile` - Update profile

---

### 4.2 Subscription Plans

**Get Available Plans:**

```
GET /v1/subscriptions/plans
```

**Response:**
```json
{
  "plans": [
    {
      "subscriptionId": "sub_1",
      "name": "Basic",
      "price": 9.99,
      "currency": "USD",
      "validity": "1 month"
    },
    {
      "subscriptionId": "sub_2",
      "name": "Premium",
      "price": 99.99,
      "currency": "USD",
      "validity": "1 year"
    }
  ]
}
```

**Subscribe to Plan:**

```
POST /v1/subscriptions/subscribe
```

**Request Body:**
```json
{
  "userId": "user_123",
  "subscriptionId": "sub_2"
}
```

---

### 4.3 Search Videos

```
GET /v1/videos/search?term={keyword}
```

**Response:**
```json
{
  "videos": [
    {
      "videoId": "vid_123",
      "title": "Stranger Things",
      "thumbnail": "https://cdn.example.com/thumb.jpg",
      "genre": "Sci-Fi",
      "duration": "3600s"
    }
  ],
  "pagination": {
    "page": 1,
    "totalPages": 10
  }
}
```

⚠️ **Important:** Use pagination!

---

### 4.4 Get Video Metadata

```
GET /v1/videos/{videoId}
```

**Response:**
```json
{
  "videoId": "vid_123",
  "title": "Stranger Things - S01E01",
  "description": "A young boy disappears...",
  "genre": "Sci-Fi",
  "duration": "3600s",
  "thumbnail": "https://cdn.example.com/thumb.jpg",
  "cast": ["Millie Bobby Brown", "Finn Wolfhard"],
  "releaseDate": "2016-07-15"
}
```

---

### 4.5 Play Video

```
GET /v1/videos/{videoId}/play
```

**Response:**
```json
{
  "manifestUrl": "https://cdn.example.com/vid_123/manifest.m3u8"
}
```

⚠️ **Key Point:** We return a **manifest file**, not the video itself!

---

## 5. High-Level Design (HLD)

### 5.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  200M Global Users                       │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │  API Gateway    │
                │                 │
                │ - Auth/AuthZ    │
                │ - Routing       │
                │ - Rate Limiting │
                └────────┬────────┘
                         │
        ┌────────────────┼────────────────┬──────────┐
        │                │                │          │
        ▼                ▼                ▼          ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐ ┌──────────┐
  │  User    │    │  Search  │    │Subscrip  │ │   Play   │
  │ Service  │    │ Service  │    │  -tion   │ │ Service  │
  └────┬─────┘    └────┬─────┘    │ Service  │ └────┬─────┘
       │               │           └────┬─────┘      │
       ▼               ▼                │            │
  ┌──────────┐    ┌──────────┐         │            │
  │  User    │    │Elastic   │         │            │
  │   DB     │    │ Search   │         │            │
  │ (MySQL)  │    │          │         │            │
  └──────────┘    └────▲─────┘         │            │
                       │               │            │
                       │ CDC           │            │
                       │               │            │
                  ┌────┴─────┐         │            │
                  │  Video   │◄────────┘            │
                  │   DB     │                      │
                  │(MongoDB) │                      │
                  └────▲─────┘                      │
                       │                            │
                       │                            │
                  ┌────┴─────┐                      │
                  │ Uploader │                      │
                  │ Service  │                      │
                  │(Backend) │                      │
                  └────┬─────┘                      │
                       │                            │
                       ▼                            │
                  ┌──────────┐                      │
                  │    S3    │◄─────────────────────┘
                  │  Bucket  │
                  │          │
                  │ - Videos │
                  │ - Chunks │
                  │ - Images │
                  └──────────┘
```

### 5.2 Service Responsibilities

**1. User Service:**
- Registration, login, logout
- JWT token generation
- User profile management

**2. Subscription Service:**
- Show available plans
- Process payments
- Update subscription status

**3. Search Service:**
- Query Elasticsearch
- Return video results

**4. Play Service:**
- Return manifest file
- Handle video playback requests

**5. Uploader Service (Backend Team Only):**
- Upload videos to S3
- Upload metadata to Video DB
- Trigger video processing

---

## 6. Low-Level Design (LLD)

### 6.1 User Service

**Database:** MySQL

**Schema:**
```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    name VARCHAR(255),
    email VARCHAR(255) UNIQUE,
    password_hash VARCHAR(255),
    subscription_status VARCHAR(50),  -- active, expired, cancelled
    subscription_expiry TIMESTAMP,
    created_at TIMESTAMP
);
```

**Flow:**
1. User registers → Store in MySQL
2. User logs in → Validate credentials → Return JWT token
3. JWT token used in header for all API calls

---

### 6.2 Subscription Service

**Database 1: Subscription DB (MySQL)**

**Schema:**
```sql
CREATE TABLE subscriptions (
    subscription_id UUID PRIMARY KEY,
    name VARCHAR(100),  -- Basic, Premium, Family
    price DECIMAL(10, 2),
    currency VARCHAR(3),
    validity VARCHAR(50),  -- 1 month, 1 year
    features JSON  -- HD, 4K, number of screens
);
```

**Database 2: Payment DB (MySQL)**

**Schema:**
```sql
CREATE TABLE payments (
    payment_id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(user_id),
    subscription_id UUID,
    amount DECIMAL(10, 2),
    currency VARCHAR(3),
    status VARCHAR(50),  -- success, failed, pending
    transaction_id VARCHAR(255),  -- From payment gateway
    created_at TIMESTAMP
);
```

**Flow:**

```
User selects plan
    │
    ▼
Subscription Service
    │
    ▼
Payment Service
    │
    ▼
Payment Gateway (Razorpay, Stripe)
    │
    ▼
Payment Success/Failure
    │
    ▼
Publish event to Kafka
    │
    ├─> Notification Consumer → Send email
    │
    └─> User Update Consumer → Update user DB
```

**Why Kafka?**
- **Decoupled:** Payment service doesn't directly update user DB
- **Reliable:** Guaranteed message delivery
- **Asynchronous:** No blocking

---

### 6.3 Search Service

**Problem:** Searching in MongoDB is slow!

**Solution:** Elasticsearch

```
Video DB (MongoDB)
    │
    │ CDC Pipeline
    ▼
Elasticsearch
    │
    │ Search Queries
    ▼
Search Service
```

**Why Elasticsearch?**
- **Fast:** Sub-second search
- **Full-text search:** "strang" finds "Stranger Things"
- **Filters:** By genre, year, rating

---

### 6.4 Video Storage & Metadata

**Video Database (MongoDB):**

**Why MongoDB?**
- **Flexible schema:** Movies vs TV shows have different fields
- **Document storage:** JSON format for metadata

**Schema:**
```json
{
  "videoId": "vid_123",
  "title": "Stranger Things - S01E01",
  "description": "A young boy disappears...",
  "genre": "Sci-Fi",
  "duration": 3600,
  "releaseDate": "2016-07-15",
  "cast": ["Millie Bobby Brown", "Finn Wolfhard"],
  "thumbnails": ["https://cdn.example.com/thumb1.jpg"],
  "manifestFile": "https://s3.example.com/vid_123/manifest.m3u8"
}
```

**S3 Bucket (Blob Storage):**

Stores:
1. **Original videos** (uploaded by backend team)
2. **Processed video chunks** (2-10 second segments)
3. **Thumbnails** (images)

---

### 6.5 Video Processing Pipeline (The Core!)

This is the **MOST IMPORTANT** part of OTT design!

#### **Challenge: Why Can't We Stream Full Video?**

**Problem:**
- 4K movie (1 hour) = **40 GB**
- 2.5 hour movie = **100 GB**
- **Impossible** to download 100 GB in real-time!
- Even with fast internet, initial load would take minutes

**Solution:** Adaptive Bitrate Streaming

---

#### **Video Processing Steps**

```
Backend Team Uploads Video
    │
    ▼
S3 Bucket (Original Video)
    │
    ▼
┌─────────────────────────────────────────┐
│  Step 1: CHUNKER SERVICE                │
│  - Splits video into 2-10 second chunks │
│  - Creates manifest file                │
└────────────────┬────────────────────────┘
                 │
                 ▼
            Kafka Broker
                 │
                 ▼
┌─────────────────────────────────────────┐
│  Step 2: VIDEO ENCODER SERVICE          │
│  - Encodes each chunk in multiple       │
│    resolutions (480p, 720p, 1080p, 4K)  │
│  - Uses H.264/H.265 codec               │
│  - Saves in MP4 or TS format            │
└────────────────┬────────────────────────┘
                 │
                 ▼
        S3 Bucket (Processed Chunks)
                 │
                 ▼
            CDN (CloudFront)
```

---

#### **Step 1: Chunker Service**

**What it does:**
- Splits 1-hour video into **360 chunks** (10 seconds each)
- Creates **manifest file** (playlist)

**Example:**
```
Original Video: movie.mp4 (1 hour)
    ↓
Chunks:
- chunk_001.mp4 (0:00 - 0:10)
- chunk_002.mp4 (0:10 - 0:20)
- chunk_003.mp4 (0:20 - 0:30)
- ...
- chunk_360.mp4 (59:50 - 60:00)
```

**Manifest File Created:**
```
manifest.m3u8 (HLS) or manifest.mpd (DASH)
```

---

#### **Step 2: Video Encoder Service**

**What it does:**
- Takes each chunk
- Encodes in **4 resolutions:**
  - 480p (low quality, 1 Mbps)
  - 720p (medium, 3 Mbps)
  - 1080p (high, 5 Mbps)
  - 4K (ultra, 15 Mbps)

**Example:**
```
chunk_001.mp4
    ↓
Encoded versions:
- chunk_001_480p.mp4
- chunk_001_720p.mp4
- chunk_001_1080p.mp4
- chunk_001_4k.mp4
```

**Video Encoding Details:**
- **Codec:** H.264 or H.265
- **Format:** MP4 (HLS) or TS (DASH)
- **Bitrate:** Varies by resolution

---

#### **Manifest File Explained**

**What is a Manifest File?**
- **Playlist** of video chunks
- Tells player **which chunks to fetch** and **in what order**
- Contains **multiple quality levels**

**Example Manifest (HLS - M3U8):**

```m3u8
#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=1000000,RESOLUTION=854x480
480p/playlist.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=3000000,RESOLUTION=1280x720
720p/playlist.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080
1080p/playlist.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=15000000,RESOLUTION=3840x2160
4k/playlist.m3u8
```

**480p Playlist:**
```m3u8
#EXTM3U
#EXTINF:10.0
chunk_001_480p.mp4
#EXTINF:10.0
chunk_002_480p.mp4
#EXTINF:10.0
chunk_003_480p.mp4
...
```

---

### 6.6 Play Service - How Video Streaming Works

#### **Flow:**

```
User clicks "Play"
    │
    ▼
Play Service
    │
    ▼
Fetch manifest file from Video DB (or Redis cache)
    │
    ▼
Return manifest to client
    │
    ▼
Client (Video Player)
    │
    ├─> Calculate bandwidth (internet speed)
    │
    ├─> Choose quality (480p, 720p, 1080p, 4K)
    │
    └─> Fetch chunks from CDN one by one
```

---

#### **Adaptive Bitrate Streaming (The Magic!)**

**How it works:**

**Time: 0 seconds**
- User clicks play
- Client checks bandwidth: **2 Mbps**
- Decision: Start with **720p**
- Fetch: `chunk_001_720p.mp4` from CDN

**Time: 6 seconds (I-Frame)**
- User watching chunk 1
- Client rechecks bandwidth: **2 Mbps** (still same)
- Decision: Continue with **720p**
- Fetch: `chunk_002_720p.mp4`

**Time: 12 seconds**
- User watching chunk 2
- Client rechecks bandwidth: **10 Mbps** (improved!)
- Decision: Upgrade to **1080p**
- Fetch: `chunk_003_1080p.mp4`

**Time: 18 seconds**
- User watching chunk 3
- Client rechecks bandwidth: **1 Mbps** (dropped!)
- Decision: Downgrade to **480p**
- Fetch: `chunk_004_480p.mp4`

**Result:** **Zero buffering!** Quality adjusts automatically.

---

#### **I-Frame (Key Frame)**

**What is an I-Frame?**
- **Checkpoint** in video chunk
- Typically at **6 seconds** in a 10-second chunk
- When player reaches I-frame, it fetches **next chunk**

**Why 6 seconds?**
- Gives **4 seconds buffer** to download next chunk
- Prevents buffering

---

### 6.7 CDN (Content Delivery Network)

**Problem:**
- S3 bucket is in **one location** (e.g., US East)
- User in India → High latency to fetch from US

**Solution:** CDN (CloudFront, Akamai)

```
User in India
    │
    ▼
CDN Edge Server (Mumbai)
    │
    ├─> Cache Hit → Return chunk (< 50ms)
    │
    └─> Cache Miss → Fetch from S3 → Cache → Return
```

**Benefits:**
- **Low latency:** Serve from nearest location
- **High availability:** Multiple edge servers
- **Reduced S3 load:** Most requests served from cache

**What's cached in CDN?**
- **Popular videos** (trending shows)
- **Recently watched** chunks
- **Regional preferences** (Bollywood in India, K-Drama in Korea)

---

### 6.8 Optimizations

#### **1. Redis Cache for Manifest Files**

**Problem:**
- Every "Play" request fetches manifest from MongoDB
- High load on database

**Solution:**
```
Play Service
    │
    ▼
Redis Cache (Check first)
    │
    ├─> Cache Hit → Return manifest
    │
    └─> Cache Miss → Fetch from MongoDB → Cache → Return
```

**TTL:** 1 hour (manifests rarely change)

---

#### **2. Pre-fetching Next Chunks**

**Optimization:**
- While playing chunk 1, **pre-fetch chunk 2 and 3**
- Smoother playback
- No waiting between chunks

---

## 7. Streaming Protocols

### 7.1 HLS (HTTP Live Streaming)

**Developed by:** Apple

**Manifest Format:** `.m3u8`

**Segment Format:** `.ts` or `.mp4`

**Codec:** H.264, H.265

**Segment Size:** 6-10 seconds

**Bitrate Switching:** **One segment at a time**

**Prefetching:** **Not supported**

**Used by:** Apple devices, Safari

---

### 7.2 DASH (Dynamic Adaptive Streaming over HTTP)

**Developed by:** Industry standard (MPEG)

**Manifest Format:** `.mpd`

**Segment Format:** `.mp4` or `.ts`

**Codec:** H.264, H.265

**Segment Size:** 2-4 seconds

**Bitrate Switching:** **Multiple segments at once**

**Prefetching:** **Supported** (can fetch 2-3 segments ahead)

**Used by:** Netflix, YouTube, Amazon Prime

---

### 7.3 Comparison

| Feature | HLS | DASH |
|---------|-----|------|
| Manifest | `.m3u8` | `.mpd` |
| Segment | `.ts`, `.mp4` | `.mp4`, `.ts` |
| Segment Size | 6-10 sec | 2-4 sec |
| Prefetch | ❌ No | ✅ Yes (2-3 segments) |
| Bitrate Switch | One at a time | Multiple at once |
| Used By | Apple, Safari | Netflix, YouTube |

**Recommendation:** **DASH** (better performance, prefetching)

---

## 8. Complete Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    Backend Team                              │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
                  ┌──────────────┐
                  │  Uploader    │
                  │  Service     │
                  └──────┬───────┘
                         │
          ┌──────────────┼──────────────┐
          │                             │
          ▼                             ▼
    ┌──────────┐                  ┌──────────┐
    │ Video DB │                  │    S3    │
    │(MongoDB) │                  │  Bucket  │
    │          │                  │          │
    │ Metadata │                  │ Original │
    └──────────┘                  │  Videos  │
                                  └────┬─────┘
                                       │
                                       ▼
                                ┌──────────────┐
                                │   Chunker    │
                                │   Service    │
                                └──────┬───────┘
                                       │
                                       ▼
                                  Kafka Broker
                                       │
                                       ▼
                                ┌──────────────┐
                                │    Video     │
                                │   Encoder    │
                                │              │
                                │ - 480p       │
                                │ - 720p       │
                                │ - 1080p      │
                                │ - 4K         │
                                └──────┬───────┘
                                       │
                                       ▼
                                ┌──────────────┐
                                │      S3      │
                                │   (Chunks)   │
                                └──────┬───────┘
                                       │
                                       ▼
                                ┌──────────────┐
                                │     CDN      │
                                │ (CloudFront) │
                                └──────▲───────┘
                                       │
                                       │
┌──────────────────────────────────────┼───────────────────────┐
│                                      │                        │
│                              200M Users                       │
│                                      │                        │
└──────────────────────────────────────┼───────────────────────┘
                                       │
                                       ▼
                              ┌─────────────────┐
                              │  API Gateway    │
                              └────────┬────────┘
                                       │
                ┌──────────────────────┼──────────────────┐
                │                      │                  │
                ▼                      ▼                  ▼
          ┌──────────┐          ┌──────────┐      ┌──────────┐
          │   User   │          │  Search  │      │   Play   │
          │ Service  │          │ Service  │      │ Service  │
          └────┬─────┘          └────┬─────┘      └────┬─────┘
               │                     │                  │
               ▼                     ▼                  ▼
          ┌──────────┐          ┌──────────┐      ┌──────────┐
          │ User DB  │          │Elastic   │      │  Redis   │
          │ (MySQL)  │          │ Search   │      │  Cache   │
          └──────────┘          └──────────┘      └──────────┘
```

---

## 9. Interview-Style Q&A

### Q1: Why not stream the full video file directly?

**A:** 
- 4K movie (1 hour) = **40 GB**
- Impossible to download in real-time
- Initial load would take **minutes**
- **Solution:** Chunk into 10-second segments (40 MB each)

### Q2: How does adaptive bitrate streaming work?

**A:**
1. Client checks bandwidth every 6 seconds (at I-frame)
2. If bandwidth high → fetch higher quality chunk
3. If bandwidth low → fetch lower quality chunk
4. **Result:** No buffering, quality adjusts automatically

### Q3: What is a manifest file?

**A:**
- **Playlist** of video chunks
- Contains multiple quality levels (480p, 720p, 1080p, 4K)
- Client uses it to decide which chunks to fetch
- **Formats:** `.m3u8` (HLS) or `.mpd` (DASH)

### Q4: Why use CDN instead of S3 directly?

**A:**
- **Latency:** CDN serves from nearest edge location
- **S3:** Single location (e.g., US East)
- **Example:** User in India → 200ms from S3, 20ms from CDN
- **Cost:** CDN caching reduces S3 bandwidth costs

### Q5: HLS vs DASH - which to use?

**A:**

**Use DASH:**
- ✅ Better performance (2-4 sec segments vs 6-10 sec)
- ✅ Prefetching supported
- ✅ Used by Netflix, YouTube
- ✅ Industry standard

**Use HLS:**
- ✅ Better Apple device support
- ✅ Safari compatibility

**Recommendation:** **DASH** for most cases

### Q6: How to handle video upload by backend team?

**A:**
- **Uploader Service** (not exposed to users)
- Backend team uploads:
  1. Original video → S3
  2. Metadata → MongoDB
- **Trigger:** Chunker service starts processing
- **Time:** Can take hours (not real-time like YouTube)

### Q7: What if CDN cache misses?

**A:**
1. CDN checks local cache → **Miss**
2. CDN fetches from S3
3. CDN caches the chunk
4. CDN returns to user
5. **Next request:** Cache hit (fast!)

### Q8: How to optimize Play Service?

**A:**
- **Redis cache** for manifest files
- **TTL:** 1 hour
- **Benefit:** Reduce MongoDB load
- **Flow:** Play Service → Redis → (if miss) → MongoDB

### Q9: Why MongoDB for video metadata?

**A:**
- **Flexible schema:** Movies vs TV shows have different fields
- **Document storage:** JSON format
- **Example:** Movie has "director", TV show has "seasons"

### Q10: Security considerations?

**A:**
- **DRM (Digital Rights Management):** Prevent piracy
- **Signed URLs:** Time-limited access to S3/CDN
- **JWT tokens:** User authentication
- **HTTPS:** Encrypted communication

---

## 10. Key Takeaways

✅ **Chunking** is essential (2-10 second segments)  
✅ **Adaptive bitrate** prevents buffering  
✅ **Manifest files** guide the player  
✅ **CDN** reduces latency dramatically  
✅ **DASH** is better than HLS (prefetching, smaller segments)  
✅ **Redis cache** for manifest files  
✅ **Elasticsearch** for fast search  
✅ **Event-driven** architecture for subscription updates  

---

## Summary

**Architecture Highlights:**
- 4 microservices (User, Subscription, Search, Play)
- 3 databases (MySQL x2, MongoDB x1)
- Elasticsearch for search
- Kafka for event-driven subscription
- S3 for video storage
- CDN for low-latency delivery
- Redis for manifest caching

**Video Processing:**
- Chunker: Splits video into 10-second segments
- Encoder: Creates 4 resolutions (480p, 720p, 1080p, 4K)
- Manifest: Playlist of chunks
- Adaptive streaming: Quality adjusts to bandwidth

**Performance:**
- Initial load: < 2 seconds
- Buffering: Near zero (adaptive bitrate)
- Search: < 500ms (Elasticsearch)

**Scale:**
- 200M users globally
- 10K videos
- CDN edge servers worldwide

**End of Lecture 4**
